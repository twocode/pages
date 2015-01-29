---
title: QEMU (Aarch64) Walkthrough
date: 2014-03-11 10:00
layout: post
category: projects
comments: true
tags: qemu arm c binary_translation
description: "
Recently I have been using Qemu-arm64 for reference. But it treated my 'msr' instruction as 'unallocated instruction'. This post records walks through Qemu and how the bug can be fixed.
" 
---


I have been using [Qemu-arm64](https://github.com/susematz/qemu/tree/aarch64-1.6) ported by SUSE for reference recently. The porting went quite well for most instructions. But a bug was exposed when it handles `msr` instruction which functions to set the `nzcv` values of `pstate` register from general register. This post introduces roughly how Qemu is organized and how the bug is fixed.

The most important folders invovled to implement new binary translation guests and instructions would be `target-[ARCH]`, which contain functions to interpret/translate guest instructions to `tcg` (Tiny Code Generator, "backend for a c compiler", quoted from Bellard, the genius author of Qemu). `linux-user` is another important folder that contains codes for Qemu user mode, including elf loader, memory/cpu initialization, etc.. The `main()` entry also resides in here as well. Other files and folders are for runtime allocation, platform emulating and supportive interfaces.

Many macros are defined and referred in `main()` to initialize differenct kinds of guest applications, including `tcg_exec_init()`, `cpu_exec_init_all()` and `cpu_init()`. Then it calls `loader_exec()` to load the guest app and copies the running parameters to it. After all the work is done, it calls `cpu_loop()` and never returns.

`cpu_loop()` is defined like an overloaded function name with different parameters. In `cpu_loop(CPUARMState *env)` where ARM guest apps are executed, it calls `cpu_arm_exec(env)` to disas(semble), translate and run the guest app. You will not locate this function by pressing `ctl+]` if you use *ctags* because the function derives from `cpu_exec`: 

    ./target-arm/cpu.h:769:#define cpu_exec cpu_arm_exec
    
which is implemented in `cpu-exec.c` and commented as "main execution loop". 

In `cpu_exec()`, Qemu sets conditional flags and initializes `TranslationBlock` at the first beginning before using `sigsetjmp()` to prepare for exception handlings. After declaring all the exception handlers, Qemu now officially begins its translation process.

Qemu stores translation blocks to caches for future reference, such that if the same guest code block appears, Qemu would first call `tb_find_fast(env)` to look up the current translation entry in the cache. If the block has been translated, there would be no need to translate again. If not found, it calls `tb_find_slow()` inside the fast loop up function to make the inquiry by using physical mappings. If neither returns the translated code from caches, Qemu calls `tb_gen_code()` to do the binary translation work. And this is also where `cpu_gen_code()` is called, which is implemented in `translate-all.c`, marking the Qemu world and translation world has been bridged.

`cpu_gen_code()` calls `gen_intermediate_code()` which is implemented by all `target-[ARCH]/translate.c` as an interface determined by running configure script. As described before, Qemu translates guest code to tcg instead of to host codes direcly for easiness for porting. Qemu catogrized three types of modes - aarch64, thumb, arm here and goes into `disas_a64_insn()` if it is aarch64 architecture.

`disas` in `disas_a64_insn()` means `disassembly` and is implemented by calling `arm_ldl_code()` to return the `insn` to interpret/translate, followed by a big round of `switch - case` sentences to handle different kinds of opcodes.

After all the description above of Qemu mechanisms, finally we come to the spot where `msr` instruction is handled:

```c
else if (get_bits(insn, 20, 12) == 0xd51) {
    handle_msr(s, insn);
    break;
}
```

According to ARM V8 manual, there are many conditions with `msr` instruction which operates the `pstate` register and setting values of `nzcv` is merely one of them working at `EL0`. When `op0` is 3, `crn` is 4 and `op2` is 0, the instruction means writing values `[31:28]` of `Xd` to `nzcv`. So we need to get the 64 bit value of `Xd`, truncate it to 32 bits, shift right 28 bits, and store it to `pstate` value:

```c
Index: target-arm/translate-a64.c
===================================================================
--- target-arm/translate-a64.c  (revision 14129)
+++ target-arm/translate-a64.c  (revision 14130)
@@ -394,9 +394,15 @@
     int crn = get_bits(insn, 12, 4);
     int op2 = get_bits(insn, 5, 3);

-    /* XXX what are these? */
-    if (op0 == 3 && op1 == 3 && op2 == 2 && !crm && crn == 13) {
-        tcg_gen_st_i64(cpu_reg(dest), cpu_env, offsetof(CPUARMState, sr.tpidr_el0));
+    /* Instructions for accessing special-purpose registers */
+    if (op0 == 3 && crn == 4 && op2 == 0) {
+        TCGv_i32 t2 = tcg_temp_new_i32();
+        tcg_gen_trunc_i64_i32(t2, cpu_reg(dest));
+        tcg_gen_shri_i32(t2, t2, 28);
+        tcg_gen_st_i64(t2, cpu_env, offsetof(CPUARMState, pstate));
+        tcg_temp_free_i32(t2);
+    } else if (op0 == 3 && op1 == 3 && op2 == 2 && !crm && crn == 13) {
+        tcg_gen_st32_i64(cpu_reg(dest), cpu_env, offsetof(CPUARMState, sr.tpidr_el0));
     } else if (op0 == 3 && op1 == 3 && (op2 == 0 || op2 == 1) && crm == 4 && crn == 4) {
         /* XXX this is wrong! */
         tcg_gen_st32_i64(cpu_reg(dest), cpu_env,
@@ -405,6 +411,7 @@
         fprintf(stderr, "MSR: %d %d %d %d %d\n", op0, op1, op2, crm, crn);
         unallocated_encoding(s);
     }
```

This patch will generate correct intermediate codes with the help of tcg. The code will then be translated into host code (x86) and ran. This will continue the walk through I described before untill it meets a branch instruction or supervisor call which marks the end of the blocks. 

<br />
