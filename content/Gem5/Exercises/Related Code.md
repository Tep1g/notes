## Exercise 1
### Program
#### `sum.c`
Referenced in the [Source Code](https://notes.tepig.pro/Gem5/Exercises/Exercise-1#source-code) section.
```c
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>

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
#### `sum_fverbose.asm`
Referenced in the [Compilation](https://notes.tepig.pro/Gem5/Exercises/Exercise-1#compilation) section.
```asm
	.file	"sum.c"
# GNU C17 (Debian 12.2.0-14) version 12.2.0 (x86_64-linux-gnu)
#	compiled by GNU C version 12.2.0, GMP version 6.2.1, MPFR version 4.1.1-p1, MPC version 1.3.1, isl version isl-0.25-GMP

# warning: MPFR header version 4.1.1-p1 differs from library version 4.2.0.
# GGC heuristics: --param ggc-min-expand=100 --param ggc-min-heapsize=131072
# options passed: -mtune=generic -march=x86-64 -O0 -fasynchronous-unwind-tables
	.text
	.type	sum_arr, @function
sum_arr:
.LFB6:
	.cfi_startproc
	pushq	%rbp	#
	.cfi_def_cfa_offset 16
	.cfi_offset 6, -16
	movq	%rsp, %rbp	#,
	.cfi_def_cfa_register 6
	movq	%rdi, -24(%rbp)	# arr, arr
	movl	%esi, %eax	# arr_len, tmp88
	movq	%rdx, -40(%rbp)	# sum, sum
	movb	%al, -28(%rbp)	# tmp89, arr_len
# sum.c:6:     for (int i = 0; i < arr_len; i++) {
	movl	$0, -4(%rbp)	#, i
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
# sum.c:6:     for (int i = 0; i < arr_len; i++) {
	addl	$1, -4(%rbp)	#, i
.L2:
# sum.c:6:     for (int i = 0; i < arr_len; i++) {
	movzbl	-28(%rbp), %eax	# arr_len, _6
	cmpl	%eax, -4(%rbp)	# _6, i
	jl	.L3	#,
# sum.c:9: }
	nop	
	nop	
	popq	%rbp	#
	.cfi_def_cfa 7, 8
	ret	
	.cfi_endproc
.LFE6:
	.size	sum_arr, .-sum_arr
	.section	.rodata
.LC0:
	.string	"%d"
	.text
	.globl	main
	.type	main, @function
main:
.LFB7:
	.cfi_startproc
	pushq	%rbp	#
	.cfi_def_cfa_offset 16
	.cfi_offset 6, -16
	movq	%rsp, %rbp	#,
	.cfi_def_cfa_register 6
	subq	$16, %rsp	#,
# sum.c:12:     uint8_t arr[] = {6, 12, 1, 4};
	movl	$67177478, -5(%rbp)	#, arr
# sum.c:13:     uint8_t arr_len = sizeof(arr) / sizeof(uint8_t);
	movb	$4, -1(%rbp)	#, arr_len
# sum.c:16:     sum_arr(arr, arr_len, &sum);
	movzbl	-1(%rbp), %ecx	# arr_len, _1
	leaq	-6(%rbp), %rdx	#, tmp87
	leaq	-5(%rbp), %rax	#, tmp88
	movl	%ecx, %esi	# _1,
	movq	%rax, %rdi	# tmp88,
	call	sum_arr	#
# sum.c:17:     printf("%d", sum);
	movzbl	-6(%rbp), %eax	# sum, sum.0_2
	movzbl	%al, %eax	# sum.0_2, _3
	movl	%eax, %esi	# _3,
	leaq	.LC0(%rip), %rax	#, tmp89
	movq	%rax, %rdi	# tmp89,
	movl	$0, %eax	#,
	call	printf@PLT	#
# sum.c:19:     return 0;
	movl	$0, %eax	#, _9
# sum.c:20: }
	leave	
	.cfi_def_cfa 7, 8
	ret	
	.cfi_endproc
.LFE7:
	.size	main, .-main
	.ident	"GCC: (Debian 12.2.0-14) 12.2.0"
	.section	.note.GNU-stack,"",@progbits

```
### `sum.asm`
Referenced in the [Compilation](https://notes.tepig.pro/Gem5/Exercises/Exercise-1#compilation) section.
```asm

sum:     file format elf64-x86-64


Disassembly of section .init:

0000000000001000 <_init>:
    1000:	48 83 ec 08          	sub    $0x8,%rsp
    1004:	48 8b 05 c5 2f 00 00 	mov    0x2fc5(%rip),%rax        # 3fd0 <__gmon_start__@Base>
    100b:	48 85 c0             	test   %rax,%rax
    100e:	74 02                	je     1012 <_init+0x12>
    1010:	ff d0                	call   *%rax
    1012:	48 83 c4 08          	add    $0x8,%rsp
    1016:	c3                   	ret

Disassembly of section .plt:

0000000000001020 <printf@plt-0x10>:
    1020:	ff 35 ca 2f 00 00    	push   0x2fca(%rip)        # 3ff0 <_GLOBAL_OFFSET_TABLE_+0x8>
    1026:	ff 25 cc 2f 00 00    	jmp    *0x2fcc(%rip)        # 3ff8 <_GLOBAL_OFFSET_TABLE_+0x10>
    102c:	0f 1f 40 00          	nopl   0x0(%rax)

0000000000001030 <printf@plt>:
    1030:	ff 25 ca 2f 00 00    	jmp    *0x2fca(%rip)        # 4000 <printf@GLIBC_2.2.5>
    1036:	68 00 00 00 00       	push   $0x0
    103b:	e9 e0 ff ff ff       	jmp    1020 <_init+0x20>

Disassembly of section .plt.got:

0000000000001040 <__cxa_finalize@plt>:
    1040:	ff 25 9a 2f 00 00    	jmp    *0x2f9a(%rip)        # 3fe0 <__cxa_finalize@GLIBC_2.2.5>
    1046:	66 90                	xchg   %ax,%ax

Disassembly of section .text:

0000000000001050 <_start>:
    1050:	31 ed                	xor    %ebp,%ebp
    1052:	49 89 d1             	mov    %rdx,%r9
    1055:	5e                   	pop    %rsi
    1056:	48 89 e2             	mov    %rsp,%rdx
    1059:	48 83 e4 f0          	and    $0xfffffffffffffff0,%rsp
    105d:	50                   	push   %rax
    105e:	54                   	push   %rsp
    105f:	45 31 c0             	xor    %r8d,%r8d
    1062:	31 c9                	xor    %ecx,%ecx
    1064:	48 8d 3d 19 01 00 00 	lea    0x119(%rip),%rdi        # 1184 <main>
    106b:	ff 15 4f 2f 00 00    	call   *0x2f4f(%rip)        # 3fc0 <__libc_start_main@GLIBC_2.34>
    1071:	f4                   	hlt
    1072:	66 2e 0f 1f 84 00 00 	cs nopw 0x0(%rax,%rax,1)
    1079:	00 00 00 
    107c:	0f 1f 40 00          	nopl   0x0(%rax)

0000000000001080 <deregister_tm_clones>:
    1080:	48 8d 3d 91 2f 00 00 	lea    0x2f91(%rip),%rdi        # 4018 <__TMC_END__>
    1087:	48 8d 05 8a 2f 00 00 	lea    0x2f8a(%rip),%rax        # 4018 <__TMC_END__>
    108e:	48 39 f8             	cmp    %rdi,%rax
    1091:	74 15                	je     10a8 <deregister_tm_clones+0x28>
    1093:	48 8b 05 2e 2f 00 00 	mov    0x2f2e(%rip),%rax        # 3fc8 <_ITM_deregisterTMCloneTable@Base>
    109a:	48 85 c0             	test   %rax,%rax
    109d:	74 09                	je     10a8 <deregister_tm_clones+0x28>
    109f:	ff e0                	jmp    *%rax
    10a1:	0f 1f 80 00 00 00 00 	nopl   0x0(%rax)
    10a8:	c3                   	ret
    10a9:	0f 1f 80 00 00 00 00 	nopl   0x0(%rax)

00000000000010b0 <register_tm_clones>:
    10b0:	48 8d 3d 61 2f 00 00 	lea    0x2f61(%rip),%rdi        # 4018 <__TMC_END__>
    10b7:	48 8d 35 5a 2f 00 00 	lea    0x2f5a(%rip),%rsi        # 4018 <__TMC_END__>
    10be:	48 29 fe             	sub    %rdi,%rsi
    10c1:	48 89 f0             	mov    %rsi,%rax
    10c4:	48 c1 ee 3f          	shr    $0x3f,%rsi
    10c8:	48 c1 f8 03          	sar    $0x3,%rax
    10cc:	48 01 c6             	add    %rax,%rsi
    10cf:	48 d1 fe             	sar    %rsi
    10d2:	74 14                	je     10e8 <register_tm_clones+0x38>
    10d4:	48 8b 05 fd 2e 00 00 	mov    0x2efd(%rip),%rax        # 3fd8 <_ITM_registerTMCloneTable@Base>
    10db:	48 85 c0             	test   %rax,%rax
    10de:	74 08                	je     10e8 <register_tm_clones+0x38>
    10e0:	ff e0                	jmp    *%rax
    10e2:	66 0f 1f 44 00 00    	nopw   0x0(%rax,%rax,1)
    10e8:	c3                   	ret
    10e9:	0f 1f 80 00 00 00 00 	nopl   0x0(%rax)

00000000000010f0 <__do_global_dtors_aux>:
    10f0:	f3 0f 1e fa          	endbr64
    10f4:	80 3d 1d 2f 00 00 00 	cmpb   $0x0,0x2f1d(%rip)        # 4018 <__TMC_END__>
    10fb:	75 2b                	jne    1128 <__do_global_dtors_aux+0x38>
    10fd:	55                   	push   %rbp
    10fe:	48 83 3d da 2e 00 00 	cmpq   $0x0,0x2eda(%rip)        # 3fe0 <__cxa_finalize@GLIBC_2.2.5>
    1105:	00 
    1106:	48 89 e5             	mov    %rsp,%rbp
    1109:	74 0c                	je     1117 <__do_global_dtors_aux+0x27>
    110b:	48 8b 3d fe 2e 00 00 	mov    0x2efe(%rip),%rdi        # 4010 <__dso_handle>
    1112:	e8 29 ff ff ff       	call   1040 <__cxa_finalize@plt>
    1117:	e8 64 ff ff ff       	call   1080 <deregister_tm_clones>
    111c:	c6 05 f5 2e 00 00 01 	movb   $0x1,0x2ef5(%rip)        # 4018 <__TMC_END__>
    1123:	5d                   	pop    %rbp
    1124:	c3                   	ret
    1125:	0f 1f 00             	nopl   (%rax)
    1128:	c3                   	ret
    1129:	0f 1f 80 00 00 00 00 	nopl   0x0(%rax)

0000000000001130 <frame_dummy>:
    1130:	f3 0f 1e fa          	endbr64
    1134:	e9 77 ff ff ff       	jmp    10b0 <register_tm_clones>

0000000000001139 <sum_arr>:
    1139:	55                   	push   %rbp
    113a:	48 89 e5             	mov    %rsp,%rbp
    113d:	48 89 7d e8          	mov    %rdi,-0x18(%rbp)
    1141:	89 f0                	mov    %esi,%eax
    1143:	48 89 55 d8          	mov    %rdx,-0x28(%rbp)
    1147:	88 45 e4             	mov    %al,-0x1c(%rbp)
    114a:	c7 45 fc 00 00 00 00 	movl   $0x0,-0x4(%rbp)
    1151:	eb 24                	jmp    1177 <sum_arr+0x3e>
    1153:	48 8b 45 d8          	mov    -0x28(%rbp),%rax
    1157:	0f b6 08             	movzbl (%rax),%ecx
    115a:	8b 45 fc             	mov    -0x4(%rbp),%eax
    115d:	48 63 d0             	movslq %eax,%rdx
    1160:	48 8b 45 e8          	mov    -0x18(%rbp),%rax
    1164:	48 01 d0             	add    %rdx,%rax
    1167:	0f b6 00             	movzbl (%rax),%eax
    116a:	8d 14 01             	lea    (%rcx,%rax,1),%edx
    116d:	48 8b 45 d8          	mov    -0x28(%rbp),%rax
    1171:	88 10                	mov    %dl,(%rax)
    1173:	83 45 fc 01          	addl   $0x1,-0x4(%rbp)
    1177:	0f b6 45 e4          	movzbl -0x1c(%rbp),%eax
    117b:	39 45 fc             	cmp    %eax,-0x4(%rbp)
    117e:	7c d3                	jl     1153 <sum_arr+0x1a>
    1180:	90                   	nop
    1181:	90                   	nop
    1182:	5d                   	pop    %rbp
    1183:	c3                   	ret

0000000000001184 <main>:
    1184:	55                   	push   %rbp
    1185:	48 89 e5             	mov    %rsp,%rbp
    1188:	48 83 ec 10          	sub    $0x10,%rsp
    118c:	c7 45 fb 06 0c 01 04 	movl   $0x4010c06,-0x5(%rbp)
    1193:	c6 45 ff 04          	movb   $0x4,-0x1(%rbp)
    1197:	0f b6 4d ff          	movzbl -0x1(%rbp),%ecx
    119b:	48 8d 55 fa          	lea    -0x6(%rbp),%rdx
    119f:	48 8d 45 fb          	lea    -0x5(%rbp),%rax
    11a3:	89 ce                	mov    %ecx,%esi
    11a5:	48 89 c7             	mov    %rax,%rdi
    11a8:	e8 8c ff ff ff       	call   1139 <sum_arr>
    11ad:	0f b6 45 fa          	movzbl -0x6(%rbp),%eax
    11b1:	0f b6 c0             	movzbl %al,%eax
    11b4:	89 c6                	mov    %eax,%esi
    11b6:	48 8d 05 47 0e 00 00 	lea    0xe47(%rip),%rax        # 2004 <_IO_stdin_used+0x4>
    11bd:	48 89 c7             	mov    %rax,%rdi
    11c0:	b8 00 00 00 00       	mov    $0x0,%eax
    11c5:	e8 66 fe ff ff       	call   1030 <printf@plt>
    11ca:	b8 00 00 00 00       	mov    $0x0,%eax
    11cf:	c9                   	leave
    11d0:	c3                   	ret

Disassembly of section .fini:

00000000000011d4 <_fini>:
    11d4:	48 83 ec 08          	sub    $0x8,%rsp
    11d8:	48 83 c4 08          	add    $0x8,%rsp
    11dc:	c3                   	ret

```
### Simulation
Code snippets referenced in the [Simulation](https://notes.tepig.pro/Gem5/Exercises/Exercise-1#simulation) section.
#### `simple.py`
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