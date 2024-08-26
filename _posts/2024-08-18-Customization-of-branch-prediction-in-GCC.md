---
layout: post
title: Branch Prediction Customization with GCC
date: 2024-08-18
description: example of branch prediction with customizations
tags:
categories:
giscus_comments:
related_posts:
---

Found an interesting way while coding... some backstory

## What is branch prediction in computer architecture?

Pipelining is an implementation technique which overlaps multiple instructions in execution. It takes advantage of parallelism that exists
among the actions needed to execute an instruction. All processors use pipelining to overlap instructions to improve performance.

Every instruction in RISC architecture can be executed in at most 5 clock cycles. These cycles are :

For simplicity, let's consider implementation of RISC instruction set for understanding the pipelining.
Every instruction in RISC set is implemented in at most 5 clock cycles. Which are, fetch, decode, execute, memory access and write back.

Simple 5 stage risc pipeline can be seen as below:

1. Instruction Fetch
2. Instruction Decode
3. Execute
4. Memory Access
5. Write Back

This can be implemented in pipelining architecture as below:

But what happens when there is if else branch?
In such cases CPU tries to do the prediction on which branch to take in order to execute the branch.

| Instruction number | Clock 1 | Clock 2 | Clock 3 | Clock 4 | Clock 5 | Clock 6 | Clock 7 | Clock 8 | Clock 9 |
| :----------------- | :-----: | :-----: | :-----: | :-----: | :-----: | :-----: | :-----: | :-----: | :-----: |
| Instruction i      |   IF    |   ID    |   EX    |   MEM   |   WB    |         |         |         |         |
| Instruction i+1    |         |   IF    |   ID    |   EX    |   MEM   |   WB    |         |         |         |
| Instruction i+2    |         |         |   IF    |   ID    |   EX    |   MEM   |   WB    |         |         |
| Instruction i+3    |         |         |         |   IF    |   ID    |   EX    |   MEM   |   WB    |         |
| Instruction i+4    |         |         |         |         |   IF    |   ID    |   EX    |   MEM   |   WB    |

But there are few limitations to this approach. One of such limitation comes while handling branches and control statements.
In control statements, the program counter can jump and instruction in pipeline can no longer be valid.
If the pipeline flushes, then the instruction will take more cycles for instructions to execute.

To avoid this pipeline flush compiler uses optimization techniques to predict which instruction the branch will take.

There are several ways by which compiler predicts these instructions. One of such example is likey and unlikely instruction.

Likely and unlikely instructions help processor decide which branch is likely to happen and thus reduces the pipeline flushes.

To test this, here is one example of the code:

```c
#include<stdio.h>

#define likely(x)       __builtin_expect(!!(x), 1)
#define unlikely(x)     __builtin_expect(!!(x), 0)

int square(int num) {
	if(num<0)
		return 0;
	return num*num;
}

int square_likely(int num) {
	if(unlikely(num<0))
		return 0;
	return num*num;
}

int main(int argc, char *argv[]) {
 	int i=0;

	i=square(i);
	printf("%d\n",i);
	i=square_likely(i);
	printf("%d\n",i);

	return 0;


}
```

In above code, `likely` and `unlikely` keywords are defined by using `__builtin_expect`.

The assembly code on x86-64 (AMD64) architecture was as follows:

```assembly
test_likely.o:     file format elf64-x86-64


Disassembly of section .text:

0000000000000000 <square>:
   0:	f3 0f 1e fa          	endbr64
   4:	31 c0                	xor    %eax,%eax
   6:	85 ff                	test   %edi,%edi
   8:	78 05                	js     f <square+0xf>
   a:	89 f8                	mov    %edi,%eax
   c:	0f af c7             	imul   %edi,%eax
   f:	c3                   	ret

0000000000000010 <square_likely>:
  10:	f3 0f 1e fa          	endbr64
  14:	89 f8                	mov    %edi,%eax
  16:	85 ff                	test   %edi,%edi
  18:	78 06                	js     20 <square_likely+0x10>
  1a:	0f af c7             	imul   %edi,%eax
  1d:	c3                   	ret
  1e:	66 90                	xchg   %ax,%ax
  20:	31 c0                	xor    %eax,%eax
  22:	c3                   	ret

Disassembly of section .text.startup:

0000000000000000 <main>:
   0:	f3 0f 1e fa          	endbr64
   4:	53                   	push   %rbx
   5:	48 8d 1d 00 00 00 00 	lea    0x0(%rip),%rbx        # c <main+0xc>
			8: R_X86_64_PC32	.LC0-0x4
   c:	31 d2                	xor    %edx,%edx
   e:	bf 02 00 00 00       	mov    $0x2,%edi
  13:	48 89 de             	mov    %rbx,%rsi
  16:	31 c0                	xor    %eax,%eax
  18:	e8 00 00 00 00       	call   1d <main+0x1d>
			19: R_X86_64_PLT32	__printf_chk-0x4
  1d:	48 89 de             	mov    %rbx,%rsi
  20:	31 d2                	xor    %edx,%edx
  22:	bf 02 00 00 00       	mov    $0x2,%edi
  27:	31 c0                	xor    %eax,%eax
  29:	e8 00 00 00 00       	call   2e <main+0x2e>
			2a: R_X86_64_PLT32	__printf_chk-0x4
  2e:	31 c0                	xor    %eax,%eax
  30:	5b                   	pop    %rbx
  31:	c3                   	ret
```

Here the function square was using simple `if` branch, but function square_likely was using `if` with `unlikely`.
We can see the difference in instruction `xor    %eax,%eax`. This instruction assigns flushes the eax register, thus return value can be 0.
This instruction in `square` function is on the line 3, where as it's on the second last line if we use `unlikely`.

Similarly another example we can consider:

```c
#include<stdio.h>
#include<time.h>

#define likely(x)       __builtin_expect(!!(x), 1)
#define unlikely(x)     __builtin_expect(!!(x), 0)

int square(int num) {
        return num*num;
}

int main(int argc, char *argv[]) {
        int i=square(time(NULL)%100);
        if (i)
                printf("value of i %d\n", i);
        puts("returning now\n");
        return 0;
}
```

Here if we use condition as `if(i)` the assembly generated in Arm architecture is:

```assembly

test_likely.o:	file format mach-o arm64

Disassembly of section __TEXT,__text:

0000000000000000 <ltmp0>:
       0: 1b007c00     	mul	w0, w0, w0
       4: d65f03c0     	ret

0000000000000008 <_main>:
       8: d10083ff     	sub	sp, sp, #32
       c: a9017bfd     	stp	x29, x30, [sp, #16]
      10: 910043fd     	add	x29, sp, #16
      14: d2800000     	mov	x0, #0
      18: 94000000     	bl	0x18 <_main+0x10>
		0000000000000018:  ARM64_RELOC_BRANCH26	_time
      1c: d29ae168     	mov	x8, #55051
      20: f2ae1468     	movk	x8, #28835, lsl #16
      24: f2c147a8     	movk	x8, #2621, lsl #32
      28: f2f47ae8     	movk	x8, #41943, lsl #48
      2c: 9b487c08     	smulh	x8, x0, x8
      30: 8b000108     	add	x8, x8, x0
      34: d37ffd09     	lsr	x9, x8, #63
      38: d346fd08     	lsr	x8, x8, #6
      3c: 0b090108     	add	w8, w8, w9
      40: 52800c89     	mov	w9, #100
      44: 1b098108     	msub	w8, w8, w9, w0
      48: 340000c8     	cbz	w8, 0x60 <_main+0x58>
      4c: 1b087d08     	mul	w8, w8, w8
      50: f90003e8     	str	x8, [sp]
      54: 90000000     	adrp	x0, 0x0 <_main+0x4c>
		0000000000000054:  ARM64_RELOC_PAGE21	l_.str
      58: 91000000     	add	x0, x0, #0
		0000000000000058:  ARM64_RELOC_PAGEOFF12	l_.str
      5c: 94000000     	bl	0x5c <_main+0x54>
		000000000000005c:  ARM64_RELOC_BRANCH26	_printf
      60: 90000000     	adrp	x0, 0x0 <_main+0x58>
		0000000000000060:  ARM64_RELOC_PAGE21	l_.str.1
      64: 91000000     	add	x0, x0, #0
		0000000000000064:  ARM64_RELOC_PAGEOFF12	l_.str.1
      68: 94000000     	bl	0x68 <_main+0x60>
		0000000000000068:  ARM64_RELOC_BRANCH26	_puts
      6c: 52800000     	mov	w0, #0
      70: a9417bfd     	ldp	x29, x30, [sp, #16]
      74: 910083ff     	add	sp, sp, #32
      78: d65f03c0     	ret
```

But if we use the instruction `if(unlikely(i))` the assembly generated is:

```assembly

test_likely.o:	file format mach-o arm64

Disassembly of section __TEXT,__text:

0000000000000000 <ltmp0>:
       0: 1b007c00     	mul	w0, w0, w0
       4: d65f03c0     	ret

0000000000000008 <_main>:
       8: d10083ff     	sub	sp, sp, #32
       c: a9017bfd     	stp	x29, x30, [sp, #16]
      10: 910043fd     	add	x29, sp, #16
      14: d2800000     	mov	x0, #0
      18: 94000000     	bl	0x18 <_main+0x10>
		0000000000000018:  ARM64_RELOC_BRANCH26	_time
      1c: d29ae168     	mov	x8, #55051
      20: f2ae1468     	movk	x8, #28835, lsl #16
      24: f2c147a8     	movk	x8, #2621, lsl #32
      28: f2f47ae8     	movk	x8, #41943, lsl #48
      2c: 9b487c08     	smulh	x8, x0, x8
      30: 8b000108     	add	x8, x8, x0
      34: d37ffd09     	lsr	x9, x8, #63
      38: d346fd08     	lsr	x8, x8, #6
      3c: 0b090108     	add	w8, w8, w9
      40: 52800c89     	mov	w9, #100
      44: 1b098108     	msub	w8, w8, w9, w0
      48: 35000108     	cbnz	w8, 0x68 <_main+0x60>
      4c: 90000000     	adrp	x0, 0x0 <_main+0x44>
		000000000000004c:  ARM64_RELOC_PAGE21	l_.str.1
      50: 91000000     	add	x0, x0, #0
		0000000000000050:  ARM64_RELOC_PAGEOFF12	l_.str.1
      54: 94000000     	bl	0x54 <_main+0x4c>
		0000000000000054:  ARM64_RELOC_BRANCH26	_puts
      58: 52800000     	mov	w0, #0
      5c: a9417bfd     	ldp	x29, x30, [sp, #16]
      60: 910083ff     	add	sp, sp, #32
      64: d65f03c0     	ret
      68: 1b087d08     	mul	w8, w8, w8
      6c: f90003e8     	str	x8, [sp]
      70: 90000000     	adrp	x0, 0x0 <_main+0x68>
		0000000000000070:  ARM64_RELOC_PAGE21	l_.str
      74: 91000000     	add	x0, x0, #0
		0000000000000074:  ARM64_RELOC_PAGEOFF12	l_.str
      78: 94000000     	bl	0x78 <_main+0x70>
		0000000000000078:  ARM64_RELOC_BRANCH26	_printf
      7c: 17fffff4     	b	0x4c <_main+0x44>
```

We can see the difference in printf function here. The printf function is pushed after the instruction puts in second example when `unlikely` keyword was used.

These are just small examples but in bigger codebase, using these likely and unlikely keywords can improve the performance significantly

## Commands used to generate the assembly code are:

1. `gcc -c -O3 -std=gnu11 test_likely.c`
   gcc command is used to compile the code and generate the code in machine level language. O3 flag shows that optimization level is 3. There are total 4 levels of optimization levels in gcc (from 0-3) with third being highest.
2. `objdump -dr test_likely.o`
   objdump command does object dump and parameters dr are used to disassemble the machine code in assembly.

## References

1. Book : Computer Architecture: A Quantitative Approach (The Morgan Kaufmann Series in Computer Architecture and Design) 5th edition
2. https://stackoverflow.com/questions/109710/how-do-the-likely-unlikely-macros-in-the-linux-kernel-work-and-what-is-their-ben
