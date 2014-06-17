---
title: XCode Debugger Bug with FPCR Register in Calling Aarch64 Assembly Function
date: 2014-06-16 16:00
layout: post
category: projects
comments: true
tags: arm XCode aarch64 A7 assembly
description: "A wierd bug related to XCode debugger when trying to call a function written in Aarch64 assembly. The debugger failed to log the correct value of Floating-Point control register, with a specific but normal instruction pattern." 
---

I was using Objective-C to call this Aarch64 code sequence, incapsulated in a separate assembly file along side the .m source file. Several breakpoints were scattered in the .m file because it seemed not applicable to set breakpoints in assembly file. Values of Vector and Floating-Point (VFP) registers, Floating-Point Control Register (FPCR) and Floating-Point Status Register (FPSR) were out of expectations. OS X version 10.8.5, XCode version 5.1, IOS 7.0, with hardware Apple Ipad Air A7 processor (ARMv8 ISA).

The entry of assembly file was exposed with a label and externed in .m file. The targeted asm code block was composed of a series of initialization instructions, which use *msr* to initialize PSTATE register and FPCR/FPSR registers, and *mov* or *fmov* instructions to initialize general-purpose registers and floating-point registers. Before entering the function, we stopped at the breakpoint and watched the initial register values.

![1.png]({{BASE_PATH}}/images/xcode_bug/1.png)

After executing the code block (specifically with the instruction: *msr fpcr, x0*), the results from debugger is quite wierd because it was not even close to the expected value stored in x0. No other instructions would change the value of FPCR. 

![2.png]({{BASE_PATH}}/images/xcode_bug/2.png)

![3.png]({{BASE_PATH}}/images/xcode_bug/3.png)

![4.png]({{BASE_PATH}}/images/xcode_bug/4.png)

I did a bisection search to the code block and located to this pattern:

![5.png]({{BASE_PATH}}/images/xcode_bug/5.png)

commenting the two lines will make debugger print FPCR as 0 while otherwise the previous strange value, which, with more inspection, was exactly the data stored right here:

![6.png]({{BASE_PATH}}/images/xcode_bug/6.png)

Therefore, the first bug is exposed: **FPCR register value was replaced by some other unrelated instruction values**.

Wait. If the register is flushed by another instruction in the debugger, than why commenting the interfering code pattern makes FPCR display 0? That should be the value stored in x0. In order to root cause the problem, I used an inline assembly code to manualy copies FPCR register value to a general-purpose register:

![7.png]({{BASE_PATH}}/images/xcode_bug/7.png)

![8.png]({{BASE_PATH}}/images/xcode_bug/8.png)

The two are different! Yes, debugger's print is not the same as its real value. And that is the second bug: **FPCR register is not correctly output in debugger window**.

The two bugs were not even close to the total number of bugs to be exposed. As a matter of fact, the debugger prints the VFP registers as double (64 bits), while it in fact is 128-bits long in Aarch64. 

![9.png]({{BASE_PATH}}/images/xcode_bug/9.png)

<br />

**Misc**

There is a lot of major differeces between GAS assembly syntax and Clang (Xcode version). The most inconvenient ones for me are two directives unsupported by Clang - *ldr Xt, =0x12345678abcdef* and *.inst efac1234*. In GCC, the compiler automatically converts the former pattern to:

    ldr Xt, label
    ...
    label:
        .word 0x78abcdef
    label2:
        .word 0x123456

and decodes the *.inst* value to coresponding disassembly code.

It is applicable to write a script to re-jigger the assembly code from the objdump-ed code from GAS syntax into Clang format. But that's not always the solution to close-source tools (compilers). I also noticed the latter incompatibility - *.inst* directive handling, had been fixed late in llvm last year, but XCode has yet to apply the features and patches.

<br />
