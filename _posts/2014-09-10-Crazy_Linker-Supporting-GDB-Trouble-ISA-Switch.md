---
title: crazy_linker Supporting GDB Debugging User Library Causing Trouble to ISA Context-Switch
date: 2014-09-10 16:00
layout: post
category: projects
comments: true
tags: kernel arm x86 android linker gdb
description: crazy_linker is a user-land linker for APP itself to load libraries themselves. It provides substitutes for dlfcn.h-providing dlxxx familiy functions like dlopen(), dlsym(), etc. Once APP uses crazy_linker (or other linker framework for self-loading), its behaviors are not accessible to the kernel.

---

Crazy_linker is a custom dynamic linker for Android program that provides similar functionalities as `/system/bin/linker` with several differences. More info can be referred in its [README](https://chromium.googlesource.com/android_tools/+/2ddbdf33f440ef5b39440053e6ca747093183814/ndk/sources/android/crazy_linker/README.TXT) for detailed information. Source code can be found [here](https://chromium.googlesource.com/android_tools/+/2ddbdf33f440ef5b39440053e6ca747093183814/ndk/sources/android/crazy_linker/README.TXT).

Crazy_linker is more powerful than system linker in some ways to make dynamic linking more customizable, like library paths abbreviation, fixed loading memory, etc. And users can add more functionalities to crazy_linker for specific purposes. Crazy_linker also supports stack unwinding to support GDB. However, In the scenario of running one ABI-supported APP which contains crazy_linker as the self-loader on a differenct platform with different ABI. It has a problem that leads to crash, caused by crazy_linker's support for GDB debugging.

Usually, when an Android binary translator runs the ARM-compiled native APP on x86, it needs to recognize/records the APIs that needs to be mapped or translated to x86 instructions. Otherwise, the APP will run into an ARM instruction that wil definitedly crash on the x86 platform. However, it is also possible that the APP run into x86 address space from ARM address space when the ARM context is being translated, and this will cause the next x86 instructions to be continued being translated, causing the APP crashing.

Crazy_linker provides GDB debugging for user libraries. It uses its DT_DEBUG entry in dynamic sections of /system/bin/app_process executable, and retrieves its function pointer of d_ptr:

```c
//dynamic section definition:
typedef struct {
        Elf32_Sword    d_tag;
        union {
               Elf32_Word     d_val;
               Elf32_Addr     d_ptr;
        } d_un;
} Elf32_Dyn;

//crazy_linker makes use of DT_DEBUG section
if (dynamic_section->d_tag == DT_DEBUG) {
…
r_debug_ = reinterpret_cast<r_debug*>(dyn_section->d_un.d_ptr);
…
}

//Later, it calls its member function pointer
r_debug_ -> r_brk();
```

r_brk is an x86 function because app_process binary is x86. API mapping can not be hooked and mapped and APP crashes from ARM space to x86 space.


<br />

