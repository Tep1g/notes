## Background Info
Before starting the exercise, we need to understand how gem5 actually works. The following questions are raised at this point:
- What architectures does gem5 simulate?
- How are hardware components represented?
- How do we actually run a simulation?

Gem5 features simulation support for several ISAs including x86, ARM, RISC-V, MIPS, POWER, and SPARC, it even supports the AMD VEGA GPU. The hardware components for the respective ISAs are written and compiled in C++, and a Python wrapper is used to configure and instantiate them during simulation.
## Program
### Source Code
Now that we understand how gem5 works, we need to create a program that our simulated hardware can actually run. This program takes the sum of the elements within an array and prints it. [The full sum.c program can be found here](Gem5/Exercises/Related-Code#Exercise-1#sumc).
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
We can use `sum_fverbose.asm` to get an idea of how the low-level instructions work. [The full sum_fverbose.asm file can be found here](Gem5/Exercises/Related-Code#Exercise-1#sum_commentedasm)
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

Alternatively, we can use objdump to analyze the compiled binary file's assembly instructions by running the following command.
```bash
 >> objdump -d sum > sum.asm
```
I've also added my own comments. [The full sum.asm can be found here](Gem5/Exercises/Related-Code#sumasm).
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

We can then use the following command to compile `sum.c` into a binary file that the simulator can run, we'll name this file `sum`.
```bash
 >> gcc -fverbose-asm -O0 sum.c -o sum
```
## Simulation
### Configuration
Since the gem5 python components are modularized, we can easily create our own configuration script. [The full simple.py script can be found here](Gem5/Exercises/Related-Code#simplepy).
#### simple.py
```python
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

The `MESITwoLevelCacheHierarchy` object allows us to define a Cache as well as its L1 and L2 size and set associativity. The `SingleChannelDDR4_2400` object represents a single 4 GB stick of DDR4 random memory access that runs at 2400 MHz. Our `SimpleProcessor` object represents a single-core CPU. We then map these components to the `SimpleBoard` object and set the clock frequency to 3GHz.

Now that we have our hardware configured, we set the path to our `sum` binary file with the `BinaryResource` object and `set_se_binary_workload()` method. Finally, we instantiate a `Simulator` object with our `SimpleBoard` object and call the `run()` method to run the simulation.
### Execution
The following command runs the simulation with the `simple.py` hardware configuration.
```bash
 >> build/X86/gem5.opt configs/simple.py
```

This results in the following output in `/m5out/stats.txt`
#### stats.txt
```txt
---------- Begin Simulation Statistics ----------
simSeconds                                   0.000225                       # Number of seconds simulated (Second)
simTicks                                    224638137                       # Number of ticks simulated (Tick)
finalTick                                   224638137                       # Number of ticks from beginning of simulation (restored from checkpoints and never reset) (Tick)
simFreq                                  1000000000000                       # The number of ticks per simulated second ((Tick/Second))
hostSeconds                                      0.26                       # Real time elapsed on the host (Second)
hostTickRate                                860768264                       # The number of ticks simulated per host second (ticks/s) ((Tick/Second))
hostMemory                                    4915616                       # Number of bytes of host memory used (Byte)
simInsts                                        87380                       # Number of instructions simulated (Count)
simOps                                         171352                       # Number of ops (including micro ops) simulated (Count)
hostInstRate                                   334694                       # Simulator instruction rate (inst/s) ((Count/Second))
hostOpRate                                     656317                       # Simulator op (including micro ops) rate (op/s) ((Count/Second))
```

[This diagram](Gem5/images/config.dot.pdf) is also generated upon simulation, this visualizes the hardware that we configured in `simple.py`.