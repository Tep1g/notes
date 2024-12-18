## Purpose
This research exercise will examine how floating point math is handled on the instruction level and on the emulation level.
## Program
### Source Code
The following program allocates the values, 0 to 249 as floating point values, to the heap and sums them up.
#### floating-point.c
```c
#include <stdint.h>
#include <stdlib.h>
#include <stdio.h>

#define NUM_ELEMENTS 250

int main() {
    float *float_arr = (float *) malloc(NUM_ELEMENTS * sizeof(float));
    for (uint8_t i=0; i < NUM_ELEMENTS; i++) {
        float_arr[i] = (float)i;
    }

    float sum = 0;
    for (uint8_t i=0; i < NUM_ELEMENTS; i++) {
        sum += float_arr[i];
    }
    printf("Sum: %f\n", sum);
}
```

### Compilation
The following makefile script is created to make compilation easier.
#### Makefile
```makefile
../bin/x86/linux/floating-point: floating-point.c
	gcc -o ../bin/x86/linux/floating-point -O0 floating-point.c
```

The following command can then be used to run the makefile script and compile the program.
```bash
 >> make
```

objdump can then be used to get a look at the low level assembly instructions.
```bash
 >> objdump -d floating-point > floating-point-with-pcs
```

The objdump output assembly file reveals the PC addresses for the instructions that read the floating point values from memory and add them. These PC addresses can be used to find their corresponding debug messages.
```
11ba:	f3 0f 10 00          	movss  (%rax),%xmm0     # Reading the float value
11be:	f3 0f 10 4d f8       	movss  -0x8(%rbp),%xmm1 # Reading the current sum
11c3:	f3 0f 58 c1          	addss  %xmm1,%xmm0      # Adding the sum and float
```
## Simulation
### Configuration
The following configuration is a simple single-core system with an L1 cache.
#### simple_system.py
```python
import m5
from m5.objects import *

# Add the common scripts to our path
m5.util.addToPath("../../")

# import the caches which we made
from learning_gem5.part1.caches import *

# Create system
system = System()

# Create clock domain
system.clk_domain = SrcClockDomain()
system.clk_domain.clock = '3GHz'
system.clk_domain.voltage_domain = VoltageDomain()

# Set RAM size to 8GB
system.mem_mode = 'timing'
system.mem_ranges = [AddrRange('8GB')]

# Create a simple x86 timing CPU core
system.cpu = X86TimingSimpleCPU()

# Create memory bus
system.membus = SystemXBar()

# Create L1 cache
system.cpu.icache = L1ICache()
system.cpu.dcache = L1DCache()

# Connect CPU to L1 cache
system.cpu.icache.connectCPU(system.cpu)
system.cpu.dcache.connectCPU(system.cpu)

# Connect L1 cache to memory bus
system.cpu.icache.connectBus(system.membus)
system.cpu.dcache.connectBus(system.membus)

# Create interrupt controller
system.cpu.createInterruptController()

# Connect the parellel I/O ports
system.cpu.interrupts[0].pio = system.membus.mem_side_ports
system.cpu.interrupts[0].int_requestor = system.membus.cpu_side_ports
system.cpu.interrupts[0].int_responder = system.membus.mem_side_ports

# Connect system port to memory bus to allow for reading and writing to memory
system.system_port = system.membus.cpu_side_ports

# Create memory controller and DRAM configuration
system.mem_ctrl = MemCtrl()
system.mem_ctrl.dram = DDR4_2400_8x8()
system.mem_ctrl.dram.range = system.mem_ranges[0]
system.mem_ctrl.port = system.membus.mem_side_ports

# Set path to binary file
binary = 'tests/test-progs/floating-point/bin/x86/linux/floating-point'

# for gem5 V21 and beyond
system.workload = SEWorkload.init_compatible(binary)

# Create process with specified binary path
process = Process()
process.cmd = [binary]
system.cpu.workload = process
system.cpu.createThreads()

# Instantiate root system
root = Root(full_system = False, system = system)
exit_event = m5.instantiate()

# Begin simulation
print("Beginning simulation!")

exit_event = m5.simulate()

# Inspect system state
print('Exiting @ tick {} because {}'
      .format(m5.curTick(), exit_event.getCause()))
```

### Simulation
The following set of commands can be run to record the simulation's debug messages.
```bash
 >> script
 >> build/X86/gem5.opt --debug-flags=DRAM,Exec,Cache configs/exercises/exercise-6/simple_system.py
 >> exit
```

This debug message shows the floating point value, in the array, being read (FloatMemRead). D=0x43790000 is the IEEE 754 single precision floating point representation of 249.0, which is the last value stored in the array. This value is retrieved from the L1 cache.
```
220564215: system.cpu: T0 : 0x11ba @main+113. 2 :   MOVSS_XMM_M : ldfp   %xmm0_low, DS:[rax] : FloatMemRead :  D=0x0000000043790000 A=0x5684
220564215: system.cpu.dcache: access for ReadReq [8b684:8b687] hit state: e (M) writable: 1 readable: 1 dirty: 1 prefetched: 0 | tag: 0x11 secure: 0 valid: 1 | set: 0xda way: 0x1
220564215: system.cpu.dcache: PC 11ba
```

This debug shows the current sum being retrieved from memory. Similarly to the last debug message, this value is also retrieved from the L1 cache and encoded in accordance with the IEEE 754 standard.
```
220566213: system.cpu: T0 : 0x11be @main+117. 2 :   MOVSS_XMM_M : ldfp   %xmm1_low, SS:[rbp + 0xfffffffffffffff8] : FloatMemRead :  D=0x0000000046f13800 A=0x7fffffffece8
220566213: system.cpu.dcache: access for ReadReq [39ce8:39ceb] hit state: e (M) writable: 1 readable: 1 dirty: 1 prefetched: 0 | tag: 0x7 secure: 0 valid: 1 | set: 0x73 way: 0x1
220566213: system.cpu.dcache: PC 11be
```

This message shows the execution of the floating point addition operation. ADDSS_XMM_XMM stands for the addition of the two single scalar precision (found in the XMM registers) since the `float` datatype is 32 bit. 0x46f32a00 is the encoded final sum of the array (31125.0).
```
220567545: system.cpu: T0 : 0x11c3 @main+122. 0 :   ADDSS_XMM_XMM : maddf   %xmm0_low, %xmm0_low, %xmm1_low : SimdFloatAdd :  D=0x0000000046f32a00
```