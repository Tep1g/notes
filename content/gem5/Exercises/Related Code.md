## `sum_commented.asm`
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