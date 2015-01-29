---
title: Heap/Stack/Mmap Execution Permission Prevented by SELinux within Android 5.0
date: 2015-01-05 16:00
layout: post
category: projects
comments: true
tags: android selinux memory kernel
description: On Nexus 9, within arm64-v8a ABI and Android 5.0 system, more restrictions regarding security appeared, including memory excecution permissions that become mandatory by SELinux.

---

We all know Android has been integrated with SELinux for security protections since KitKat. SELinux set up a few rules in favor of memory protections, including memory (heap/stack/file mmap) execution limitations. On Nexus 9, these restrictions seem to be harsher.

Within a JNI applied program, a piece of memory if dynamically applied, it will be requested to have an execution permission before copy a piece of instructions/codes to it. The code will be acting like a JIT-ed code as it will be registered to one of JNI's native functions with `(*env)->RegisterNatives()` API so that it will be called in Java code.

First, we start with **malloc** which gets us the heap memory.

```C
JNINativeMethod methods1[] = {
    { "dynamicRegisteredNative", "()I", NULL },
};
...
func_jitted = (FunctionPointer)malloc(24);
memcpy(func_jitted, TargetFunctionPointer, 24);
methods1[0].fnPtr = (void *)func_jitted;
if (mprotect((char*)(((uint64_t)func_jitted) & ~(PAGESIZE-1)), PAGESIZE, PROT_EXEC | PROT_READ | PROT_WRITE) < 0) {
    printf("mprotect: %d\n", func_jitted, errno);
}
```

The logic is quite straightforward: applying 24 Bytes memory on heap, filling up its content, and register its label to JNI native function table. The magic number 24 in code means there were 6 assemblly instruction from `TargetFunctionPointer` (each instruction 4 Bytes, 6 in total). 

How ever, the JIT-ed code could not be executed. The reason from logcat was `.../* avc: denied { execheap } */...`. Looks like setting execution permission on heap is not pleased by SELinux. We have to try other methods.

Now we try enabling the same logic on **stack** with local variables. Neglecting the fact that even permission is granted, the code will not be retrieved as the stack will be gone in the fly.

    ```c
    uint32_t code[256];
    func_jitted = (FunctionPointer)code;
    ```
    
Not surprisely, the logcat shows `.../* avc: denied { execstack } */...` error.

Trying to use **zero page mmap** to set the `PROT_EXEC` at the first beginning.

    ```c
    if ((func_jitted = (FunctionPointer)mmap(NULL, 
            PAGESIZE, PROT_WRITE | PROT_READ | PROT_EXEC,
            MAP_SHARED, open("/dev/zero", O_RDWR), 0)) == MAP_FAILED) {
        printf("func_jitted: %p, mmap: %d\n", func_jitted, errno);
    }
    ```

Even though the `mmap()` function allows to pass in a *PROT_EXEC* property, the logcat still shows `.../* avc: denied { execute } for path="/dev/zero" */...`, denying our accesses to the memory page.

It should be noted that all the above attempts still cannot work with `setenforce=1` which could set SELinux to "permissive" mode.

The last effort would be using the **anonymous page**, and this time, it finally worked.

    ```c
    if ((func_jitted = (FunctionPointer)mmap(NULL, 
            24, PROT_WRITE | PROT_READ | PROT_EXEC,
            MAP_PRIVATE | MAP_ANONYMOUS, -1, 0)) == MAP_FAILED) {
        printf("mmap: %d\n", func_jitted, errno);
    }
    ```

Do not forget to specify `MAP_PRIVATE` of `MAP_SHARED` with `MAP_ANONYMOUS` and the `fd` parameter is recommended, and required on some system, to be -1 when tring to apply for an anonymous page.

To sum up, if you tend to run a piece of code during runtime dynamically, in a SELinux-featured system, you have to use anonymous page to make that work.



<br />

