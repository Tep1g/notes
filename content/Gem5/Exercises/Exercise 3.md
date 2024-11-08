## Purpose
In this exercise, we'll be simulating the parallel execution of a multithreaded program on a dual core system. The point of this exercise will be to see the difference in execution speed between row-major and column-major indexing as well as seeing what happens when each core has to share L2Cache.

## Program
### Source Code
This program will have two threads summing the values within a two dimensional array of random numbers. The size of this array is set to 2500 4 byte integers, this amounts to 10KB.
#### sum-3.c
```c
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#include <sys/time.h>

#define NUM_ROWS 50
#define NUM_COLUMNS 50

struct matrix_info {
    uint32_t **matrix;
};

void *sum_by_row_major(uint32_t **matrix) {
    struct timeval t_start, t_stop, t_result;
    gettimeofday(&t_start, NULL);
    printf("Row major thread started\n");
    uint32_t sum = 0;
    for (int i = 0; i < NUM_ROWS; i++) {
        for (int j = 0; j < NUM_COLUMNS; j++) {
            sum += matrix[i][j];
        }
    }
    gettimeofday(&t_stop, NULL);
    timersub(&t_stop, &t_start, &t_result);
    printf("Row major thread finished in %u microseconds\nSum: %u\n", (long int)t_result.tv_usec, sum);
}

void *sum_by_column_major(void *threadid) {
    struct timeval t_start, t_stop, t_result;
    gettimeofday(&t_start, NULL);
    struct matrix_info *info = (struct matrix_info *) threadid;
    printf("Column major thread started\n");
    uint32_t sum = 0;
    for (int j = 0; j < NUM_COLUMNS; j++) {
        for (int i = 0; i < NUM_ROWS; i++) {
            sum += info->matrix[i][j];
        }
    }
    gettimeofday(&t_stop, NULL);
    timersub(&t_stop, &t_start, &t_result);
    printf("Column major thread finished in %u microseconds\nSum: %u\n", (long int)t_result.tv_usec, sum);
}

int main() {
    uint32_t **matrix = (uint32_t **)malloc(NUM_ROWS * sizeof(uint32_t *));

    for (int i = 0; i < NUM_ROWS; i++) {
        matrix[i] = (uint32_t *)malloc(NUM_COLUMNS * sizeof(uint32_t));
    }

    for (int i = 0; i < NUM_ROWS; i++) {
        for (int j = 0; j < NUM_COLUMNS; j++) {
            matrix[i][j] = rand();
        }
    }
    struct matrix_info info;
    info.matrix = matrix;
    
    pthread_t row_major_thread;
    pthread_create(&row_major_thread, NULL, sum_by_column_major, (void *)&info);

    // Execute the last thread with this thread context to appease SE mode
    sum_by_row_major(matrix);

    pthread_join(row_major_thread, NULL);
    return 0;
}
```

It's important to point out line 64.
```c
	// Execute the last thread with this thread context to appease SE mode
	sum_by_row_major(matrix);
```

Syscall Emulation mode only allows n-1 threads to be executed where n is the number of cores. This is because `main()` has to be constantly running on one of the cores. To address this problem, we can execute our "thread" in `main()`.
### Compilation
To speed up the compilation process, we can create the following Makefile script.
```makefile
../bin/x86/linux/sum-3: sum-3.c
	gcc -o ../bin/x86/linux/sum-3 sum-3.c -pthread
```

Then we can run the following command while inside of the src directory of the sum-3 source file.
```bash
 >> make
```

## Simulation
### Configuration
For this configuration, we'll have to create two cores and a cache hierarchy for each one. We'll also need to make sure each core's cache hierarchy is smaller than the array that we create, the argument parser will allow us to pass a custom size to our cache.
#### multi_core_system.py
```python
import argparse
import m5
from m5.objects import *

# Add the common scripts to our path
m5.util.addToPath("../../")
from common.FileSystemConfig import config_filesystem

# import the caches which we made
from learning_gem5.part1.caches import *

# Define number of cores
NUM_CORES = 2

# Create system
system = System()

# Create clock domain
system.clk_domain = SrcClockDomain()
system.clk_domain.clock = '3GHz'
system.clk_domain.voltage_domain = VoltageDomain()

# Set RAM size to 8GB
system.mem_mode = 'timing'
system.mem_ranges = [AddrRange('8GB')]

# Create memory bus
system.membus = SystemXBar()

# Create x86 timing cores
system.cpu = [X86TimingSimpleCPU() for _ in range(NUM_CORES)]

# Create L2 buses
system.l2bus = [L2XBar() for _ in range(NUM_CORES)]

parser = argparse.ArgumentParser(description='A simple system with 2-level cache.')
parser.add_argument("--l1d_size",
                    help="L1 data cache size. Default: Default: 64kB.")
parser.add_argument("--l2_size",
                    help="L2 cache size. Default: 256kB.")

options = parser.parse_args()

# Create L2 Caches
system.l2cache = [L2Cache(options) for _ in range(NUM_CORES)]

# Repeat for each core
for i in range(NUM_CORES):

    # Create L1 cache
    system.cpu[i].icache = L1ICache()
    system.cpu[i].dcache = L1DCache(options)

    # Connect core to L1 cache
    system.cpu[i].icache.connectCPU(system.cpu[i])
    system.cpu[i].dcache.connectCPU(system.cpu[i])

    # Connect L2 bus to L1 cache
    system.cpu[i].icache.connectBus(system.l2bus[i])
    system.cpu[i].dcache.connectBus(system.l2bus[i])

    # Connect L2 cache to memory bus
    system.l2cache[i].connectCPUSideBus(system.l2bus[i])
    system.l2cache[i].connectMemSideBus(system.membus)

    # Create interrupt controller
    system.cpu[i].createInterruptController()

    # Connect the parellel I/O ports
    system.cpu[i].interrupts[0].pio = system.membus.mem_side_ports
    system.cpu[i].interrupts[0].int_requestor = system.membus.cpu_side_ports
    system.cpu[i].interrupts[0].int_responder = system.membus.mem_side_ports

# Connect system port to memory bus to allow for reading and writing to memory
system.system_port = system.membus.cpu_side_ports

# Create memory controller and DRAM configuration
system.mem_ctrl = MemCtrl()
system.mem_ctrl.dram = DDR4_2400_8x8()
system.mem_ctrl.dram.range = system.mem_ranges[0]
system.mem_ctrl.port = system.membus.mem_side_ports

# Set path to binary file
binary = 'tests/test-progs/sum-3/bin/x86/linux/sum-3'

# for gem5 V21 and beyond
system.workload = SEWorkload.init_compatible(binary)

# Create processes with specified binary path
process = Process()
process.cmd = [binary]
for cpu in system.cpu:
    cpu.workload = process
    cpu.createThreads()

# Set up the pseudo file system for the threads function above
config_filesystem(system)

# Instantiate root system
root = Root(full_system = False, system = system)
m5.instantiate()

# Begin simulation
print("Beginning simulation!")
exit_event = m5.simulate()

# Inspect system state
print('Exiting @ tick {} because {}'
      .format(m5.curTick(), exit_event.getCause()))

```

### Execution
We can pass in our custom cache sizes and run the simulation with the following terminal command.
```bash
 >> build/X86/gem5.opt configs/exercises/exercise-3/multi_core_system.py --l1d_size='2kB' --l2_size='4kB'
```

As expected, the row-major summing thread finishes faster.
```
Row major thread started
Column major thread started
Row major thread finished in 109 microseconds
Sum: 3204380660
Column major thread finished in 139 microseconds
Sum: 3204380660
```

To set the L1 and L2 cache sizes and record debug messages, we run the following set of commands.
```bash
 >> script
 >> build/X86/gem5.opt --debug-flags=DRAM,Exec,Cache configs/exercises/exercise-3/multi_core_system.py --l1d_size='2kB' --l2_size='4kB'
 >> exit
```

By reading the debug messages, we can check to see if the threads are actually running in parallel.
```
728124813: system.cpu1: T0 : 0x11fb @sum_by_row_major+114. 2 :   ADD_M_I : add   t1d, t1d, t2d : IntAlu :  D=0x0000000000000000
728124813: system.cpu1.dcache: access for WriteReq [a3ec4:a3ec7] hit state: 6d61662f (M) writable: 1 readable: 1 dirty: 1 prefetched: 0 | tag: 0x28f secure: 0 valid: 1 | set: 0xb way: 0

728124813: system.cpu0: T0 : 0x128c @sum_by_column_major+94    : sal	0x2 
```

The cores are labeled as cpus. Looking closely, we can see that `cpu1` is executing the `sum_by_row_major` thread and `cpu0` is executing the `sum_by_column_major` thread. Additionally, the debug messages tell us that these specific debug messages occur at the same tick `728124813` (same point in time). This tells us that each core and thread is actually running in parallel as we would expect.

Since our L1 and L2 cache are smaller than the heap's allocated memory size, we also expect to see some evictions within our hierarchy.

For this instruction, we see it trying to fetch data within the L1 data cache and incurring a cache miss. It then creates a read request to look for the desired data in the L2 cache.
```
728595675: system.cpu0: T0 : 0x1290 @sum_by_column_major+98. 0 :   ADD_R_R : add   rax, rax, rdx : IntAlu :  D=0x0000000000000000
728595675: system.cpu0.icache: access for ReadReq [1290:1297] IF hit state: c4ab85b6 (E) writable: 1 readable: 1 dirty: 0 prefetched: 0 | tag: 0 secure: 0 valid: 1 | set: 0x4a way: 0
728596341: system.cpu0.dcache: access for ReadReq [8ebbc:8ebbf] miss
728597007: system.cpu0.dcache: sendMSHRQueuePacket: MSHR ReadReq [8ebbc:8ebbf]
728597007: system.cpu0.dcache: createMissPacket: created ReadSharedReq [8eb80:8ebbf] from ReadReq [8ebbc:8ebbf]
```

The L2 cache receives the read request and it also incurs a cache miss so it creates a read request to look for the desired data in DRAM.
```
728597007: system.l2cache0: access for ReadSharedReq [8eb80:8ebbf] miss
728604000: system.l2cache0: sendMSHRQueuePacket: MSHR ReadSharedReq [8eb80:8ebbf]
728604000: system.l2cache0: createMissPacket: created ReadSharedReq [8eb80:8ebbf] from ReadSharedReq [8eb80:8ebbf]
```

The DRAM then receives the read request, fetches the data, and propagates it back up to the L2 cache through a read response.
```
728604000: system.mem_ctrl.dram: Address: 0x8eb80 Rank 0 Bank 7 Row 2
728604000: system.mem_ctrl.dram: Timing access to addr 0x8eb80, rank/bank/row 0 7 2
728604000: system.mem_ctrl.dram: Schedule RD/WR burst at tick 728604000
```

Once the L2 cache receives the DRAM's response, it proceeds to store the data for that address by evicting a cache block that already exists. The data is then propagated up to the L1 cache.
```
728644959: system.l2cache0: recvTimingResp: Handling response ReadResp [8eb80:8ebbf]
728644959: system.l2cache0: Block for addr 0x8eb80 being updated in Cache
728644959: system.l2cache0: Create CleanEvict CleanEvict [8bd80:8bdbf]
728644959: system.l2cache0: Block addr 0x8eb80 (ns) moving from  to state: ef6795b6 (E) writable: 1 readable: 1 dirty: 0 prefetched: 0 | tag: 0x475 secure: 0 valid: 1 | set: 0x6 way: 0x2
```

Once the L1 cache receives the L2 cache's response, it proceeds to do the same and store the data for that address by evicting a pre-existing cache block.
```
728651952: system.cpu0.dcache: recvTimingResp: Handling response ReadResp [8eb80:8ebbf]
728651952: system.cpu0.dcache: Block for addr 0x8eb80 being updated in Cache
728651952: system.cpu0.dcache: Create CleanEvict CleanEvict [8e380:8e3bf]
728651952: system.cpu0.dcache: Block addr 0x8eb80 (ns) moving from  to state: 6 (E) writable: 1 readable: 1 dirty: 0 prefetched: 0 | tag: 0x23a secure: 0 valid: 1 | set: 0xe way: 0x1
```

We then see these same cache block get evicted later in the program.
```
730981953: system.l2cache0: Create CleanEvict CleanEvict [8eb80:8ebbf]
730988946: system.cpu0.dcache: Create CleanEvict CleanEvict [8eb80:8ebbf]
```