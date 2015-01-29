---
title: Higher Memory Addressing (> 4G) in ARM64-V8a
date: 2014-10-21 16:00
layout: post
category: projects
comments: true
tags: arm aarch64 memory addressing linker relocation 64-bit
description: When Aarch64 instructions addresses a label or branch es from/to a higher space address, the compiler will perform wrong relocations and fail to link, if no specific operations involved.

---

Linking Aarch64 segments to higher address space (>4G) meets with relocation errors. I have looked into those errors in building executables (same rule can be applied to shared objects) to higher addresses.

```c
xiangyuh@hubuntu:~/ubt/Houdini/trunk/dev-tools/rit/art$ aarch64-linux-gnu-gcc -nostdlib -static -Wl,--section-start,.ritdata=0x111111111ffff800 -Wl,--section-start,.cfgdata=0x111111112fffff80 -Wl,--section-start,.exit=0x11111111300000 -Wl,--section-start,.text=0x11111111004001b8  vfp.s
/tmp/ccPAB3G2.o: In function `rit_b2_c1_end':
(.text+0x1003a0): relocation truncated to fit: R_AARCH64_ABS32 against `.cfgdata'
/tmp/ccPAB3G2.o: In function `rit_b3_end':
(.text+0x1008c8): relocation truncated to fit: R_AARCH64_JUMP26 against `.exit'
collect2: error: ld returned 1 exit status
```

The relocation table looks like:

```c
xiangyuh@hubuntu:~/ubt/Houdini/trunk/dev-tools/rit/art$ aarch64-linux-gnu-readelf --relocs vfp.o > vfp.relocs

Relocation section '.rela.text' at offset 0x164348 contains 43 entries:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
0000001003a0  001e00000102 R_AARCH64_ABS32   0000000000000000 .cfgdata + a0
…
0000001008c8  00210000011a R_AARCH64_JUMP26  0000000000000000 .exit + 0
…
```

The correspondent instructions are:

```asm
xiangyuh@hubuntu:~/ubt/Houdini/trunk/dev-tools/rit/art$ aarch64-linux-gnu-gcc -nostdlib -static -c vfp.s
…
100308:   180004ca    ldr w10, 1003a0 <rit_b2_c1_end+0xc>
…
1008c8:   14000000    b   0 <_start-0x100000>
…
```

The first error means `0x10030` stores the value that will be loaded into `w10`, and it’s only a 32-bit relocation. After changing the instruction to `ldr x10, 1003a0`, it will become a 64-bit relocation and fix the bug. I think it’s *a weird compiler behavior that determine the relocation address size by the instruction’s destination register width, as relocation handles addresses and destination register width decides number of bytes to load from some address*. They are different worlds.

The branch error is similar to the **“Branch out of range”** compiling error when trying to branch to an address that is out of range, but this one occurs in linking phase. The instruction will be patched to a relocated address which is 26-bit width range but the .exit segment actually was assigned to a high address. The fix is to use the 64-bit register as a branch register:

```asm
ldr             x0, =exit                
br              x0
```

To sum up, in Aarch64 environment, ***1. Use 64-bit width destination register when loading from relocation-required addresses like labels, functions, from higher address space. 2. When trying to branch to a higher space, use a 64-bit register as intermediate and use BR (branch register) instruction to jump to it.***


<br />

