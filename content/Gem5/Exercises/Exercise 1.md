## Background Info
Before starting the exercise, we need to understand how gem5 actually works. The following questions are raised at this point:
- What architectures does gem5 simulate?
- How are hardware components represented?
- How do we actually run a simulation?

Gem5 features simulation support for several ISAs including x86, ARM, RISC-V, MIPS, POWER, and SPARC, it even supports the AMD VEGA GPU. The hardware components for the respective ISAs are written and compiled in C++, and a Python wrapper is used to configure and instantiate them during simulation.
## Program
### Source Code
Now that we understand how gem5 works, we need to create a program that our simulated hardware can actually run. This program takes the sum of the elements within an array and prints it. The full sum.c program can be found [here](Gem5/Exercises/Related-Code#Exercise-1#sumc).
#### sum.c
```c
static void sum_arr(uint8_t *arr, uint8_t arr_len, uint8_t *sum) {
    for (int i = 0; i < arr_len; i++) {
        *sum += arr[i];
    }
}

int main() {
    uint8_t arr[] = {6, 12, 1, 4};
    uint8_t arr_len = sizeof(arr) / sizeof(uint8_t);

    uint8_t sum;
    sum_arr(arr, arr_len, &sum);
    printf("%d", sum);

    return 0;
}
```
### Compilation
The following command uses GCC to compile `sum.c` into an assembly file that we can read. The `-S` flag compiles it to assembly, `-fverbose-asm` generates comments that make it easier to understand the assembly code, and `-O0` disables compiler optimization.
```bash
 >> gcc -S -fverbose-asm -O0 sum.c -o sum_fverbose.asm
```
We can use `sum_fverbose.asm` to get an idea of how the low-level instructions work. The full sum_fverbose.asm file can be found [here](Gem5/Exercises/Related-Code#Exercise-1#sum_commentedasm)
#### sum_fverbose.asm
```asm
# sum.c:6:     for (int i = 0; i < arr_len; i++) {
	jmp	.L2	#
.L3:
# sum.c:7:         *sum += arr[i];
	movq	-40(%rbp), %rax	# sum, tmp90
	movzbl	(%rax), %ecx	# *sum_12(D), _1
# sum.c:7:         *sum += arr[i];
	movl	-4(%rbp), %eax	# i, tmp91
	movslq	%eax, %rdx	# tmp91, _2
	movq	-24(%rbp), %rax	# arr, tmp92
	addq	%rdx, %rax	# _2, _3
	movzbl	(%rax), %eax	# *_3, _4
# sum.c:7:         *sum += arr[i];
	leal	(%rcx,%rax), %edx	#, _5
	movq	-40(%rbp), %rax	# sum, tmp93
	movb	%dl, (%rax)	# _5, *sum_12(D)
```
####
Alternatively, we can use objdump to analyze the compiled binary file's assembly instructions by running the following command.
```bash
 >> objdump -d sum > sum.asm
```
I've also added my own comments. The full sum.asm can be found [here](Gem5/Exercises/Related-Code#sumasm).
#### sum.asm
```asm
0000000000001139 <sum_arr>:
    1139:	55                   	push   %rbp
    113a:	48 89 e5             	mov    %rsp,%rbp
    113d:	48 89 7d e8          	mov    %rdi,-0x18(%rbp)
    1141:	89 f0                	mov    %esi,%eax
    1143:	48 89 55 d8          	mov    %rdx,-0x28(%rbp)
    1147:	88 45 e4             	mov    %al,-0x1c(%rbp)
    114a:	c7 45 fc 00 00 00 00 	movl   $0x0,-0x4(%rbp)
    1151:	eb 24                	jmp    1177 <sum_arr+0x3e>
    1153:	48 8b 45 d8          	mov    -0x28(%rbp),%rax     # load sum pointer into tmp90
    1157:	0f b6 08             	movzbl (%rax),%ecx          # copy value, that sum pointer points to, to ecx
    115a:	8b 45 fc             	mov    -0x4(%rbp),%eax      # load i into tmp91 (array index)
    115d:	48 63 d0             	movslq %eax,%rdx            # move and sign extend tmp91 to 64-bit reg
    1160:	48 8b 45 e8          	mov    -0x18(%rbp),%rax     # load arr pointer into tmp92
    1164:	48 01 d0             	add    %rdx,%rax            # offset the array pointer by the array index
    1167:	0f b6 00             	movzbl (%rax),%eax          # load the array value to eax
    116a:	8d 14 01             	lea    (%rcx,%rax,1),%edx   # add sum and arr[i] and store the result in _5 (ecx is rcx and eax is rax)
    116d:	48 8b 45 d8          	mov    -0x28(%rbp),%rax     # load sum pointer into tmp93
    1171:	88 10                	mov    %dl,(%rax)           # store _5 to sum address (dl is LSB of edx)
    1173:	83 45 fc 01          	addl   $0x1,-0x4(%rbp)      # increment i
    1177:	0f b6 45 e4          	movzbl -0x1c(%rbp),%eax     # load arr_len to eax reg
    117b:	39 45 fc             	cmp    %eax,-0x4(%rbp)      # skip loop if counter >= arr_len
    117e:	7c d3                	jl     1153 <sum_arr+0x1a>
    1180:	90                   	nop
    1181:	90                   	nop
    1182:	5d                   	pop    %rbp
    1183:	c3                   	ret
```
####
We can then use the following command to compile `sum.c` into a binary file that the simulator can run, we'll name this file `sum`.
```bash
 >> gcc -fverbose-asm -O0 sum.c -o sum
```
## Simulation
### System Method
#### Configuration
According to the [simple config section](https://www.gem5.org/documentation/learning_gem5/part1/simple_config/) of the gem5 documentation. the traditional way of setting up a configuration script is to create a `System` object, define the low level hardware, and manually connect these low level pieces. The full simple_system.py script can be found [here](Gem5/Exercises/Related-Code#simple_systempy).
##### System
```python
# Create system
system = System()
```
The `System` object will contain all of the hardware as its members.
##### Clock
```python
# Create clock domain
system.clk_domain = SrcClockDomain()
system.clk_domain.clock = '3GHz'
system.clk_domain.voltage_domain = VoltageDomain()
```
The `SrcClockDomain` sets our CPU speed.
##### CPU
```python
# Create a simple x86 timing CPU core
system.cpu = X86TimingSimpleCPU()
```
Create a single x86 timing mode CPU core.
##### Memory
```python
# Set RAM size to 16GB
system.mem_mode = 'timing'
system.mem_ranges = [AddrRange('16GB')]

# Create memory controller and DRAM configuration
system.mem_ctrl = MemCtrl()
system.mem_ctrl.dram = DDR4_2400_8x8()
system.mem_ctrl.dram.range = system.mem_ranges[0]
system.mem_ctrl.port = system.membus.mem_side_ports

# Create memory bus
system.membus = SystemXBar()

# Connect system port to memory bus to allow for reading and writing to memory
system.system_port = system.membus.cpu_side_ports
```
We have to manually create and configure the memory and controller.
##### Cache
```python
cache_enabled = (input("Configure system with cache?: ").lower() == "yes")
if cache_enabled:
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
else:
    # Connect CPU cache ports to memory bus ports since we aren't using cache
    system.cpu.icache_port = system.membus.cpu_side_ports
    system.cpu.dcache_port = system.membus.cpu_side_ports
```
We have to manually create and create and configure the L1 and L2 cache, and connect them to the CPU.
##### PIO
```python
# Create interrupt controller
system.cpu.createInterruptController()

# Connect the parellel I/O ports
system.cpu.interrupts[0].pio = system.membus.mem_side_ports
system.cpu.interrupts[0].int_requestor = system.membus.cpu_side_ports
system.cpu.interrupts[0].int_responder = system.membus.mem_side_ports
```
The CPU's parallel I/O ports need to be hooked up to the memory bus.
##### Program File Path
```python
# Set path to binary file
binary = 'tests/test-progs/sum/bin/x86/linux/sum'

# for gem5 V21 and beyond
system.workload = SEWorkload.init_compatible(binary)
```
##### Run Simulation
```python
# Create process with specified binary path
process = Process()
process.cmd = [binary]
system.cpu.workload = process
system.cpu.createThreads()

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
#### Execution
The following command runs the simulation with the `simple.py` hardware configuration.
```bash
 >> build/X86/gem5.opt configs/exercises/exercise-1/simple_system.py
```

This results in the following prompt
```
Configure system with cache?:
```
##### Include Cache
Entering "yes" results in the following [diagram](simple_system_cache.dot.pdf) and output.
###### stats.txt
This is the information, within stats.txt, that we're more interested in.
```txt
system.cpu.numCycles                           746936                       # Number of cpu cycles simulated (Cycle)
system.cpu.cpi                               8.542757                       # CPI: cycles per instruction (core level) ((Cycle/Count))
system.cpu.ipc                               0.117058                       # IPC: instructions per cycle (core level) ((Count/Cycle))
```
##### Don't Include Cache
Entering "no" results in the following [diagram](simple_system_no_cache.dot.pdf) and output. As demonstrated below, including a cache results in almost 25x the performance of the non-cache system.
###### stats.txt
```txt
system.cpu.numCycles                         18520537                       # Number of cpu cycles simulated (Cycle)
system.cpu.cpi                             211.820632                       # CPI: cycles per instruction (core level) ((Cycle/Count))
system.cpu.ipc                               0.004721                       # IPC: instructions per cycle (core level) ((Count/Cycle))
```
### Board Method
#### Configuration
Alternatively, we can easily create our own configuration script with the `board` object since the gem5 python components are modularized. The full simple_board.py script can be found [here](Gem5/Exercises/Related-Code#simple_boardpy).
##### Cache
```python
cache_enabled = (input("Configure board with cache?: ").lower() == "yes")
if cache_enabled:
    cache_hierarchy = PrivateL1PrivateL2CacheHierarchy(
        l1d_size="16kB",
        l1i_size="64kB",
        l2_size="256kB",
    )
else:
    cache_hierarchy = NoCache()
```
The `PrivateL1PrivateL2CacheHierarchy` object represents an L1 and L2 cache hierarchy if we opt to include cache in our system
##### Memory
```python
# Give ourselves 4GB of the best DDR4 the simulator has to offer
memory = SingleChannelDDR4_2400(size="16GB")
```
The `SingleChannelDDR4_2400` object represents a single 16 GB stick of DDR4 random memory access that runs at 2400 MHz.
##### Processor
```python
# X86 CPU
processor = SimpleProcessor(cpu_type=CPUTypes.TIMING, isa=ISA.X86, num_cores=1)
```
Our `SimpleProcessor` object represents a single-core CPU.
##### Board
```python
board = SimpleBoard(
    clk_freq="3GHz",
    processor=processor,
    memory=memory,
    cache_hierarchy=cache_hierarchy,
)
```
We then map these components to the `SimpleBoard` object and set the clock frequency to 3GHz.
##### Program File Path
```python
binary = BinaryResource(local_path="tests/test-progs/sum/bin/x86/linux/sum")
board.set_se_binary_workload(binary)
```
Now that we have our hardware configured, we set the path to our `sum` binary file with the `BinaryResource` object and `set_se_binary_workload()` method. 
##### Run Simulation
```python
simulator = Simulator(board=board)
simulator.run()
```
Finally, we create a `Simulator` object with our `SimpleBoard` object and call the `run()` method to start the simulation.
#### Execution
The following command runs the simulation with the `simple.py` hardware configuration.
```bash
 >> build/X86/gem5.opt configs/exercises/exercise-1/simple_board.py
```

This results in the following prompt
```
Configure system with cache?:
```
##### Include Cache
Entering "yes" results in the following [diagram](simple_board_cache.dot.pdf) and output. Similarly to the last program, the lack of a cache significantly bottlenecks performance.
###### stats.txt
```txt
board.processor.cores.core.numCycles           475522                       # Number of cpu cycles simulated (Cycle)
board.processor.cores.core.cpi               5.438577                       # CPI: cycles per instruction (core level) ((Cycle/Count))
board.processor.cores.core.ipc               0.183872                       # IPC: instructions per cycle (core level) ((Count/Cycle))
```
##### Don't Include Cache
Entering "no" results in the following [diagram](simple_board_no_cache.dot.pdf) and output. Notably, the performance of this `board` implementation is almost identical to that of our `system` implementation.
###### stats.txt
```txt
board.processor.cores.core.numCycles         18520537                       # Number of cpu cycles simulated (Cycle)
board.processor.cores.core.cpi             211.820632                       # CPI: cycles per instruction (core level) ((Cycle/Count))
board.processor.cores.core.ipc               0.004721                       # IPC: instructions per cycle (core level) ((Count/Cycle))
```