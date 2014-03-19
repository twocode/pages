---
title: AArch64 Ldr/Str Cluster Decoder in QEMU
date: 2014-03-14 10:00
layout: post
category: projects
comments: true
tags: qemu arm c binary_translation
description: " In ARM AArch64, base memory instructions (ldr/str, ldrsb, ldrsh, ldrsw, ldrb/strb, ldrh, ldrw) are classified in the same cluster. The decoder/encoder routines are described in ARM v8 manual. But QEMU ARM64 branch cleaerly misunderstood it." 
---

In the last post, I described [Qemu-arm64](https://github.com/susematz/qemu/tree/aarch64-1.6) and how it mis-operated the `msr` instruction. But clearly it has more decoder errors than this. Now it is the memory instructions that have been decoded incorrectly.

In ARM AArch64, base memory instructions (`ldr/str, ldrsb, ldrsh, ldrsw, ldrb/strb, ldrh, ldrw`) are classified in the same cluster. The encoding/decoding routines are described in ARM v8 manual (P. 178). 

One of the the comlexites that have been added to AArch64 compared to the previous versions is the use of different size of registers, or in other words, `Wt`s and `Xt`s. Besides *literal* offseting, these base memory instructions have two ways for get addresses offsets: *register* and *immediate*; within *immediate*, indexing methods can be divided into: *pre-index*, *post-index* and *unsigned offset*. Base memory instructions are also allowed, except (`strb, ldrsw`), to operate 32-size of registers, which are those `Wt` registers, meaning the low 32 bits of the 64-bit general-purpose registers. How a piece of instruction means to work is specified in the instruction bits. 

The trickiest part is that `ldr/str` have the different encoding/decoding paradigms. That is to say, the different bits in the instruction are not used to decode the same information. Specifically for example, when `size` (`insn[31:30]`) is bigger than `0b10` or `0b11`, then it is a `ldr` or `str` instruction and `opc0 == 0` indicaes it is `ldr` and `str` otherwise. Register size is 32 if `size == 2` and 64 otherwise. However, if it is not a `ldr` or `str` instruction, `opc1` is used to determine if it is *sign-extended* and `opc0` is used to decide the register size. The two kinds are completely different encoded/decoded, and yet they reside in the same cluster. Therefore, writing the decoder in QEMU should be extremyly rigorous.

![decoder.jpg]({{BASE_PATH}}/images/0317/decoder.jpg)

decoding rule of a *pre-indexed* immediate offset instruction (`insn[11:10] == 0b11`)

In function `handle_ldst()`, it only went through the first condition and used the same decoding rule when treating the rest, thus wrong register sizes are got and some instructions are treated as *unallocated encodings*. The fix is:

    ...
     static void handle_ldst(DisasContext *s, uint32_t insn)
     {
    +    //TODO, if wback && dest = src, then unallocated or nop
         int dest = get_reg(insn);
         int source = get_bits(insn, 5, 5);
         int offset;
    @@ -1528,22 +1461,15 @@
                 unallocated_encoding(s);
                 return;
             }
    -    } else if (!opc1) {
    +    } else if (size >= 2 && !opc1) {
             regsize = (size == 3) ? 3 : 2;
    -    } else {
    -        if (size == 3) {
    -            /* prefetch */
    -            if (!is_store) {
    -                unallocated_encoding(s);
    -            }
    -            return;
    -        }
    -//        if (size == 2 && !is_store) {
    -//            unallocated_encoding(s);
    -//        }
    +    } else if (opc1){
             is_store = false;
             is_signed = true;
    -        regsize = is_store ? 3 : 2;
    +        regsize = get_bits(insn, 22, 1) ? 2 : 3;
    +    } else {
    +        is_signed = false;
    +        regsize = 2;
    ... 

Besides, `sign-extension` has a function scope regarding to different register sizes. When register size is 32 bits, it only extends in the low 32 bits of the destination registers:

    +++++++++++++++++++++++++++++++
         0        |    xxxxxxx   
    +++++++++++++++++++++++++++++++
    63            31              0

But QEMU treats it like it is a 64-bit-register-size instruction and operates the high 32 bits as well. It can be fixed:

    +    } else {
    +        if (is_signed) {
    +            /* XXX check what impact regsize has */
    +            switch (size) {
    +            case 0:
    +                {
    +                    if (regsize == 2) {
    +                        TCGv_i32 w = tcg_temp_new_i32();
    +                        tcg_gen_trunc_i64_i32(w, cpu_reg(dest));
    +                        tcg_gen_qemu_ld8s(w, tcg_addr, get_mem_index(s));
    +                        tcg_gen_extu_i32_i64(cpu_reg(dest), w);
    +                        tcg_temp_free(w);
    +                    } else {
    +                        tcg_gen_qemu_ld8s(cpu_reg(dest), tcg_addr, get_mem_index(s));
    +                    }
    +                    break;
    +                }
    +            case 1:
    +                {
    +                    if (regsize == 2) {
    +                        TCGv_i32 w = tcg_temp_new_i32();
    +                        tcg_gen_trunc_i64_i32(w, cpu_reg(dest));
    +                        tcg_gen_qemu_ld16s(w, tcg_addr, get_mem_index(s));
    +                        tcg_gen_extu_i32_i64(cpu_reg(dest), w);
    +                        tcg_temp_free(w);
    +                    } else {
    +                        tcg_gen_qemu_ld16s(cpu_reg(dest), tcg_addr, get_mem_index(s));
    +                    }
    +                    break;
    +                }
    +            case 2:
    +                tcg_gen_qemu_ld32s(cpu_reg(dest), tcg_addr, get_mem_index(s));
    +                break;
    +
    
These fixes can fix bugs in ld/st instruction decoders in QEMU.

<br />
