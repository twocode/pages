---
title: Zygote Libraries Conflict Local Libraries Having the Same Name
date: 2014-07-09 16:00
layout: post
category: projects
comments: true
tags: Android jni runtime dlopen zygote
description: "After Android has been initiated, Zygote will be running with system libraries loaded, most of which reside in /system/lib. When applications try to load libraries with the same library name, no matter through System.loadLibrary("<name>") or using dlopen() within JNI, the library will be ignored by Android linker and returns the handler of the previous loaded library in solist."

---

After Android has been initiated, Zygote will be running with system libraries loaded, most of which reside in */system/lib*. When applications try to load libraries with the same library name, no matter through *System.loadLibrary("<name>")* or using *dlopen()* within JNI, the library will be ignored by Android linker and returns the handler of the previous loaded library in *solist*.

**Android KitKat 4.4.2, Nexus 5 (ARM), Intel reference phone (x86)**

The defect can be verified through instances. 

First, use adb to shell into the Android OS and get the process id of Zygote:

    ps -ef | grep zygote

and find out which system libraries it has loaded with:

    cat /proc/<zygote_pid>/maps | grep /system/lib

Pick up one library under */system/lib* and also in the solist. Say, *libft2.so* (FreeType Project), and use it as the targeted library to be loaded.

Write a local JNI source code with the name *ft2.c* and built as *libft2.so*. Suppose the native function is called *fool_libft2()*, write another JNI library to load *libft2.so* with *dlopen()*.

Locate *fool_libft2()* with *dlsym()*, and it returns NULL. Android linker reads only the library name istead of its whole path. Thus, when the user wants to load a local library, which happens to have the same name as the system library, it will fail to locate the right one.

The feature is not developer-friendly because sometimes developers want to use previous of system libraries which have deprecated functions by the latest ones, or developers are simply not aware of the Android internals and happen to name their libraries the same as system libraries. If such cases occur, it might be difficult to root cause the problems.

**Pictures are yet to upload.**

<br />

