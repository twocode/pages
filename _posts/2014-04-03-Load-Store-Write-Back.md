---
title: Behaviors of Load/Store Instructions When WBACK is Set and Dest Register Equals to Base Register
date: 2014-04-03 10:00
layout: post
category: projects
comments: true
tags: arm Foudation_Model aarch64 isa qemu
description: "This post describes the bahavior in Load/Store instructions, when destination register (Rt) is the same as base register for addressing (Rn), although it can mostly be blamed to compilers for generating such confusing instructions whose behaviors are uncertain not standalized." 
---

There are two types of memory addressing in AArch64 (ARM) - **immediate offset** and **register offset**.

**immediate offset** can also be divided into three catagories:

*post-index*: `ldr xt, [xn], #simm`, where `xn` will be updated to `xn + #simm`, which is called *write back*, and is flagged as the `wback` bit in the instruction.

*pre-index*: `ldr xt, [xn, #simm]`, where `wback` is also set.

*unsigned offset*: `ldr xt, [xn, #pimm]`, where `wback` is **cleared**.

**register offset** has only one format of addressing and `wback` is always **cleared**.

*p.s. Regsize could be 32 to operate on wt/wn registers and #simm and #pimm have their own ranges, and I will not be that detailed here*

Behaviors of these instructions, accoring to AArch64 Manual (V8), are:

    If a Load instrctino specifies writeback and the register being loaded is also the base register, then one of following bahaviors occurs:
    
    - The instruction is UNALLOCATED.
    - The instruction is treated as a NOP.
    - The instruction performs the loadusing the specified addressing modeand the base register becomes UNKNOWN. In addition, if an exception occurs during the execution of such an instruction, the base address might be corrupted so that the instruction cannot be repeated.


**However**, When such cases/instructions appear, qemu and Foudation Model have different results on the three choices. Qemu stores the addressing value to the register (Rt == Rn) and Foundation Model stores the value to the register. Apparently the differences are caused by that *"qemu performs ld/st first, wback second; Founcation Model performs wback first, ld/st second"*. In order to make qemu behave like a real device, I modified the logic to Foundation Model (although it is not a real device, either).

<br />
It gets interesting when coming to the instruction where offset is `0`: `ldr xt, [xn]`. What is the meaning of the instruction, a *pre-index* instruction where `#simm` is `0`, or an *unsigned offset* instruction where `#pimm` is `0`? 

It turns out that compiler (*aarch64-linux-gnu* tool chain) consider it to be an *unsigned offset* memory addressing instruction, therefore its `wback` is **cleared**, and that is also the reason why qemu and Foundation Model get the same result, which is to set the register value from memory if it is Load or do nothing if otherwise.

<br />
