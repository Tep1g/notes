## Purpose
In this exercise, we'll be simulating the parallel execution of a multithreaded program on a dual core system. The point of this exercise will be to see the difference in execution speed between row-major and column-major indexing as well as seeing what happens when each core has to share L2Cache.

## Program
### Source Code
This program will have two threads summing the values within a two dimensional array of random numbers. The size of this array is set to 2500 4 byte integers, this translates to 10KB.
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