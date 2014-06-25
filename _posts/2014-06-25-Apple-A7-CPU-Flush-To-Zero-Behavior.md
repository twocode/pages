---
title: Apple A7 CPU Wrong Behavior with Flush-To-Zero (FZ)
date: 2014-06-25 16:00
layout: post
category: projects
comments: true
tags: arm aarch64 gdb qemu Apple A7 XCode
description: "According to ARMv8 manual, when FZ (Flush-to-Zeto) bit in FPCR register iset, the floating-point value will be flushed to zero. Apple A7, however, seems to fail such rule." 

---

According to ARMv8 manual, when FZ (Flush-to-Zeto) bit in FPCR register is set, the floating-point value will be flushed to zero. Apple A7, however, seems to fail such rule. The situation was observed on **FRECPE** scalar instructions.

Taking **FRECPE** as the example - `frecpe d5, d24`, where:

    [FPUSTATE] q05=87ef8159e5abfa35:822bfed276fe8b85
    [FPUSTATE] q24=97b71326e088e1a8:4008a8919f545aa0

The function of this instruction was merely getting the reciprocal of `d24 (0x4008a8919f545aa0)`. Walking through the pseudo codes in the Manual:

    // FPRecipEstimate()
    // =================
    bits(N) FPRecipEstimate(bits(N) operand, FPCRTypefpcr)
        Appendix G ARMv8 Pseudocode Library 
        G.3 Common library pseudocode
        AppxG-5030 Copyright © 2013 ARM Limited. All rights reserved. ARM DDI 0487A.a
        Non-Confidential - Beta ID090413
        assert N IN {32, 64};
    (type,sign,value) = FPUnpack(operand, fpcr);
    
    ...
    ...
    if ...
    elsif fpcr.FZ == '1'
    && ((N == 32 && Abs(value) >= 2.0^126)
            || (N == 64 && Abs(value) >= 2.0^1022)) then
        // Result flushed to zero of correct sign
        result = FPZero(sign);
        FPProcessException(FPExc_Underflow, fpcr);
    else ...
    ...
    ...
    if N == 32 then
        result = sign : result_exp<N-25:0> : fraction<51:29>;
    else // N == 64
        result = sign : result_exp<N-54:0> : fraction<51:0>;
    return result;

The value `0x4008a8919f545aa0` falls into the condition above and will have `d5` register flushed because `FZ = 1`:

    [FPUSTATE]FPCR: 05800000

QEMU implemented such rule in the `HELPER.c` as:

    float64 HELPER(recpe_f64)(float64 input, void *fpstp)
    {
        float_status *fpst = fpstp;
        ...
        ...
        /* Deal with any special cases */
        if (float64_is_any_nan(f64)) {
            ...
            ...   
        } else if (float64_is_infinity(f64)) {
            ...
        } else if (float64_is_zero(f64)) {
            ...
        } else if ((f64_val & ~(1ULL << 63)) < (1ULL << 50)) {
            /* Abs(value) < 2.0^-1024 */
            ...
        } else if (f64_exp >= 1023 && fpst->flush_to_zero) {
            float_raise(float_flag_underflow, fpst);
            return float64_set_sign(float64_zero, float64_is_neg(f64));
        }
        ...
    
        /* result = sign : result_exp<10:0> : fraction<51:0> */
        return make_float64(f64_sbit |
                            ((r64_exp & 0x7ff) << 52) |
                            r64_frac);
    }

Recompile QEMU with `-g` option and use gdb to set a breakpoint in the `else if` condigtion logic above, not only can we make sure that the value fell into this condition, but we could also watch the values:

    ------------------------------------------
       lqq/home/xiangyuh/ubt/Houdini/trunk/dev-tools/qemu-aarch64-cosim/qemu-aarch64/target-arm
       /helper.c        return nan;                                                           x
       x4692        } else if (float64_is_infinity(f64)) {                                    x
       x4693            return float64_set_sign(float64_zero, float64_is_neg(f64));           x
       x4694        } else if (float64_is_zero(f64)) {                                        x
       x4695            float_raise(float_flag_divbyzero, fpst);                              x
       x4696            return float64_set_sign(float64_infinity, float64_is_neg(f64));       x
       x4697        } else if ((f64_val & ~(1ULL << 63)) < (1ULL << 50)) {                    x
       x4698            /* Abs(value) < 2.0^-1024 */                                          x
       x4699            float_raise(float_flag_overflow | float_flag_inexact, fpst);          x
       x4700            if (round_to_inf(fpst, f64_sbit)) {                                   x
       x4701                return float64_set_sign(float64_infinity, float64_is_neg(f64));   x
       x4702            } else {                                                              x
       x4703                return float64_set_sign(float64_maxnorm, float64_is_neg(f64));    x
       x4704            }                                                                     x
       x4705        } else if (f64_exp >= 1023 && fpst->flush_to_zero) {                      x
      >x4706            float_raise(float_flag_underflow, fpst);                              x
       x4707            return float64_set_sign(float64_zero, float64_is_neg(f64));           x
       x4708        }                                                                         x
       x4709                                                                                  x
       x4710        r64 = call_recip_estimate(f64, 2045, fpst);                               x
       x4711        r64_val = float64_val(r64);                                               x
       x4712        r64_exp = extract64(r64_val, 52, 11);                                     x
    b+ x4713        r64_frac = extract64(r64_val, 0, 52);                                     x
       x4714                                                                                  x
       x4715        /* result = sign : result_exp<10:0> : fraction<51:0> */                   x
       x4716        return make_float64(f64_sbit |                                            x
       x4717                            ((r64_exp & 0x7ff) << 52) |                           x
       x4718                            r64_frac);                                            x
       x4719    }                                                                             x
       x4720                                                                                  x
       mqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqj
    multi-thre Thread 0x6252f In: helper_recpe_f64                   Line: 4706 PC: 0x6007d6f0
    warning: Invalid layout specified.
    Usage: layout prev | next | <layout_name>
    
    (gdb) p fpst
    $1 = (float_status *) 0x62547530
    (gdb) p *fpst
    $2 = {float_detect_tininess = 1 '\001', float_rounding_mode = 1 '\001',
      float_exception_flags = 41 ')', floatx80_rounding_precision = 0 '\000',
      flush_to_zero = 1 '\001', flush_inputs_to_zero = 1 '\001', default_nan_mode = 0 '\000'}
    (gdb) p f64_exp
    $3 = 1024
    (gdb)

`fpst` is the pointer to FPCR register emulated in QEMU and `flush_to_zero` flag was set. `exp` referred to the `exp` fields specified in IEEE754 and it was 1024 which led the work flow into the `flush_to_zero` condition block. ***The whole process was no problem.***

However, the same code ran on A7 produced different results. 

**Pictures are yet to upload**

To verify its excecution process, each field of value `0x4008a8919f545aa0` should be calculated. But fraction bits could not be calculated because the `recip_estimate(scaled)` function specified in the Manual:

    // Call C function to get reciprocal estimate of scaled value.
    // Input is rounded down to a multiple of 1/512.
    estimate = recip_estimate(scaled);

was implemented by platform. But the `exp` value from the control flow could be calculated according to the Manul and it turned out it was exactly the same as the last condition block in the Manual, which meant the A7 ignored the `FZ = 1` condition.

<br />

