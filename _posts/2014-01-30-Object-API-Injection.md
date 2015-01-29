---
title: C++ API Mapping from ARM to x86 Context
date: 2014-01-30 22:00
layout: post
category: projects
comments: true
tags: disassembly RE VisualStudio x86 arm c++
description: "
In out project, we load an ARM-built WP8 app (A-app) in x86-built WP8 app (X-app). One of the steps is to inject the API calls. the IAT (Imports Address Table) is modified in A-app to x86 APIs in x86 libraries. Problems might occur because of the differences between arm ARM and x86 calling conventions.
" 
---


***Background:*** *In out project, we load an ARM-built WP8 app (A-app) in x86-built WP8 app (X-app). One of the steps is to inject the API calls. the IAT (Imports Address Table) is modified in A-app to x86 APIs in x86 libraries. Problems might occur because of the differences between arm ARM and x86 calling conventions.*

When dealing with `Platform::Object::Object()` API calling, the `this` pointer was always not the right one and seemed to be a wild pointer. I did some reverse engineering and binary analysis into the different libraies, dlls, apps built on different platfroms to explore the root cause and came out a conclusion.

Normally, before calling `Platform::Object::Object()`
     
```asm
00334B73  mov         edi,dword ptr [this]
00334B76  lea         esi,[edi+8] 
00334B79  push        esi 
00334B7A  call        Platform::Object::Object (033407Ah)
(Release version)
 
0142A142  mov         eax,dword ptr [this] 
0142A145  add         eax,8 
0142A148  push        eax  
0142A149  call        Platform::Object::Object (01351D93h)
(Debug version)
```
     
In `ubt.dll` (where we do the API injection after modifying the IAT table in PE loader):
     
```asm
0255C07E  mov         dword ptr [ebp-2Ch],12345678h; 
0255C085  mov         eax,dword ptr [ebp-2Ch]      ;
0255C088  mov         eax,eax                      ;
```

*asm code inserted in the stripped binary to locate `object::object()` funcp call.*

```asm
0255C08A  mov         eax,dword ptr [ebp-0Ch] 
0255C08D  mov         eax,dword ptr [eax] 
0255C08F  mov         edx,dword ptr [ebp-28h] 
0255C092  mov         ecx,eax 
0255C094  call        edx                          ;object::object() x86 funcp
0255C096  jmp         0255C0FA                    ;leave, ret
```
     
     
`Object::Object()` in x86 (vccorelib.dll):
     
```asm
1001C331: 55                 push        ebp
1001C332: 8B EC              mov         ebp,esp
1001C334: 8B 45 08           mov         eax,dword ptr [ebp+8]
1001C337: C7 00 6C 1D 03 10  mov         dword ptr [eax],offset ??_7Object@Platform@@6B@
1001C33D: 5D                 pop         ebp
1001C33E: C3                 ret
```
     
`Object::Object()` in ARM (vccorelib.dll):
     
```asm
10019BE4: 4B01      ldr         r3,10019BEC
10019BE6: 6003      str         r3,[r0]
10019BE8: 4770      bx          lr
10019BEA: DEFE      __debugbreak
10019BEC: 1D4C 1004 dcd         ??_7Object@Platform@@6B@
```
     

From the two versions of `vccorelib.dll` libraries where `Object::Object()` lies, we find that it does not look exactly like in C++ where nothing has been passed into the function, but it has been stored on the stack. 

In ARM:

put `??_7Object@Platform@@6B@` on `[r0]`, `r0` is the parameter.
     
In x86:

put `??_7Object@Platform@@6B@` on `[eax]`, `eax` is on `[ebp+8]` (p.s. interpreted as `[this]` in debugger), `[this_previous] + 8` pushed in the caller function (`_stdcall`/`_thiscall` convention).

So I passed `context->r0` to the function and it succeeded.

<br />
