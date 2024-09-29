## Background Info
Before starting the exercise, we need to understand how gem5 actually works. The following questions are raised at this point:
- What architectures does gem5 simulate?
- How are hardware components represented?
- How do we actually run a simulation?

Gem5 features simulation support for several ISAs including x86, ARM, RISC-V, MIPS, POWER, and SPARC, it even supports the AMD VEGA GPU. The hardware components for the respective ISAs are written and compiled in C++, and a Python wrapper is used to configure and instantiate them during simulation.
## Program
### Source Code
Now that we understand how gem5 works, we need to create a program that our simulated hardware can actually run. We'll name this file [sum.c](https://notes.tepig.pro/Gem5/Exercises/Related-Code#Exercise-1#sumc) . This program takes the sum of the elements within an array and prints it.
### Compilation
The following command uses GCC to compile `sum.c` into an assembly file that we can read. The `-S` flag compiles it to assembly, `-fverbose-asm` generates comments that make it easier to understand the assembly code, and `-O0` disables compiler optimization. We'll name this file [sum_fverbose.asm](https://notes.tepig.pro/Gem5/Exercises/Related-Code#Exercise-1#sum_commentedasm).
```bash
 >> gcc -S -fverbose-asm -O0 sum.c -o sum_fverbose.asm
```

We can use `sum_commented.asm` to get an idea of how the low-level instructions work.

Then, we can use the following command to compile `sum.c` into a binary file that the simulator can run. We'll name this file `sum`.
```bash
 >> gcc -fverbose-asm -O0 sum.c -o sum
```

Additionally, we can use objdump to analyze the compiled binary file's assembly instructions by running the following command. We'll name this file [sum.asm]()
```bash
 >> objdump -d sum > sum.asm
```
Notice how the `sum.asm` instructions are simpler than `sum_commented.asm`. I've added my own comments to this file.
## Simulation
### Configuration
Since the gem5 python components are modularized, we can easily create our own configuration script.
```python
from gem5.components.boards.simple_board import SimpleBoard
from gem5.components.cachehierarchies.ruby.mesi_two_level_cache_hierarchy import (
    MESITwoLevelCacheHierarchy,
)
from gem5.components.memory.single_channel import SingleChannelDDR4_2400
from gem5.components.processors.cpu_types import CPUTypes
from gem5.components.processors.simple_processor import SimpleProcessor
from gem5.isas import ISA
from gem5.resources.resource import BinaryResource
from gem5.simulate.simulator import Simulator

cache_hierarchy = MESITwoLevelCacheHierarchy(
    l1d_size="16kB",
    l1d_assoc=8,
    l1i_size="16kB",
    l1i_assoc=8,
    l2_size="256kB",
    l2_assoc=16,
    num_l2_banks=1,
)

# Give ourselves 4GB of the best DDR4 the simulator has to offer
memory = SingleChannelDDR4_2400(size="4GB")

# X86 CPU
processor = SimpleProcessor(cpu_type=CPUTypes.TIMING, isa=ISA.X86, num_cores=1)

board = SimpleBoard(
    clk_freq="3GHz",
    processor=processor,
    memory=memory,
    cache_hierarchy=cache_hierarchy,
)

binary = BinaryResource(local_path="tests/test-progs/sum/bin/x86/linux/sum")
board.set_se_binary_workload(binary)

simulator = Simulator(board=board)
simulator.run()
```
### Execution
#### System Diagram
[The following diagram](Gem5/images/config.dot.pdf) is generated upon simulation execution.