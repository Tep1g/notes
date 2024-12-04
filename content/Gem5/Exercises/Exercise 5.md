## Purpose
This exercise will demonstrate the execution of gem5's m5 ops. m5 ops are magic instructions that can be executed by the emulated system. These instructions can perform unique actions such as switching between cores, resetting stats, and pausing a program and creating a checkpoint. Checkpoints capture the state of the system and the execution of the program, this allows for system analysis at specific points in time. Additionally, execution can be resumed from specific checkpoints which makes them handy for executing long and slow programs.

## Program
### Source Code
The following program is rather simple, it instantiates a couple of floating point variables. The program is split into 3 sections: variable initialization, a sum operation, and a multiplication operation.
#### checkpoint.c
```c
#include <stdio.h>
#include <stdint.h>
#include <gem5/m5ops.h>

int main(void) {
    float a = 4.8f;
    float b = 13.7f;
    float sum;
    float product;

    m5_reset_stats(0,0);
    sum = a+b;

    m5_checkpoint(0, 0);
    m5_reset_stats(0,0);
    product = a*b;
}
```

### Compilation
Before compiling the C program, the m5 library must first be built. The following terminal command must be run from the `util/m5/` directory.
```bash
 >> scons build/{TARGET_ISA}/out/m5
```

The m5 library must also be linked during compilation. To speed up this process, a makefile script can be created.
#### Makefile
```makefile
TARGET_ISA=x86

GEM5_HOME=$(realpath ../../../../)
$(info   GEM5_HOME is $(GEM5_HOME))

CXX=gcc

CFLAGS=-I$(GEM5_HOME)/include

LDFLAGS=-L$(GEM5_HOME)/util/m5/build/$(TARGET_ISA)/out -lm5

OBJECTS= ../bin/x86/linux/checkpoint

all: checkpoint

checkpoint:
	$(CXX) -o $(OBJECTS) -O0 checkpoint.c $(CFLAGS) $(LDFLAGS)

clean:
	rm -f $(OBJECTS)
```

Once compiled, `objdump` can be used to get the program counter addresses of each instruction.
```bash
 >> objdump -d checkpoint > checkpoint-with-pcs
```
## Simulation
### Configuration
#### simple_system.py
```python
import argparse
import m5
from m5.objects import *

# Add the common scripts to our path
m5.util.addToPath("../../")

# import the caches which we made
from learning_gem5.part1.caches import *


parser = argparse.ArgumentParser()
parser.add_argument("--run_from_checkpoint", action="store_true")
parser.add_argument("--save_checkpoint", action="store_true")
options = parser.parse_args()

# Create system
system = System()

# Create clock domain
system.clk_domain = SrcClockDomain()
system.clk_domain.clock = '3GHz'
system.clk_domain.voltage_domain = VoltageDomain()

# Set RAM size to 8GB
system.mem_mode = 'atomic'
system.mem_ranges = [AddrRange('8GB')]

# Create a simple x86 atomic CPU core
system.cpu = X86AtomicSimpleCPU()

# Create memory bus
system.membus = SystemXBar()

# Create L1 cache
system.cpu.icache = L1ICache()
system.cpu.dcache = L1DCache()

# Connect CPU to L1 cache
system.cpu.icache.connectCPU(system.cpu)
system.cpu.dcache.connectCPU(system.cpu)

# Create L2 bus
system.l2bus = L2XBar()

# Connect L2 bus to L1 cache
system.cpu.icache.connectBus(system.l2bus)
system.cpu.dcache.connectBus(system.l2bus)

# Create L2 cache and connect it to memory bus
system.l2cache = L2Cache()
system.l2cache.connectCPUSideBus(system.l2bus)
system.l2cache.connectMemSideBus(system.membus)

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
binary = 'tests/test-progs/checkpoint/bin/x86/linux/checkpoint'

# for gem5 V21 and beyond
system.workload = SEWorkload.init_compatible(binary)

# Create process with specified binary path
process = Process()
process.cmd = [binary]
system.cpu.workload = process
system.cpu.createThreads()

# Instantiate root system
root = Root(full_system = False, system = system)

if options.run_from_checkpoint:
      exit_event = m5.instantiate("checkpoint-dir")
else:
      exit_event = m5.instantiate()

# Begin simulation
print("Beginning simulation!")

exit_event = m5.simulate()

# Inspect system state
print('Exiting @ tick {} because {}'
      .format(m5.curTick(), exit_event.getCause()))

if options.save_checkpoint:
      m5.checkpoint("checkpoint-dir")
```