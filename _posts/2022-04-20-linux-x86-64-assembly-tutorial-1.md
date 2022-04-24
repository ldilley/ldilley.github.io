---
layout: post
title: Linux x86-64 Assembly Tutorial (part 1)
tags: [linux, assembly, tutorial]
comments: true
---

### Introduction
After finally getting around to playing with 64-bit assembly recently, I wanted to share the knowledge since there does not seem to be
many tutorials out there surprisingly. The available information is mostly scattered between AMD64 programming manuals, assembler
documentation, code snippets on the Web, and system call tables or Linux source code. I figured that I would consolidate all of that
information here to make things easier. If you just want to see the code, feel free to skip ahead.

### Useful Links
I recommend reading through the three AMD programming manuals below located at the
[AMD Developer Guides & Manuals](http://developer.amd.com/resources/documentation-articles/developer-guides-manuals/) site if you're
serious about getting into this stuff:

1. [AMD64 Architecture Programmer's Manual Volume 2: System Programming](http://amd-dev.wpengine.netdna-cdn.com/wordpress/media/2012/10/24593_APM_v21.pdf)
2. [AMD64 Architecture Programmer's Manual Volume 3: General Purpose and System Instructions](http://amd-dev.wpengine.netdna-cdn.com/wordpress/media/2008/10/24594_APM_v3.pdf)
3. [AMD64 Architecture Programmer's Manual Volume 1: Application Programming](http://amd-dev.wpengine.netdna-cdn.com/wordpress/media/2012/10/24592_APM_v11.pdf)

These will provide you with a lot of in-depth information about the hardware architecture.

Documentation for NASM can be found [here](http://www.nasm.us/xdoc/2.11.02/html/nasmdoc0.html).

I also recommend [Jeff Duntemann's Assembly Language Step by Step](http://www.duntemann.com/assembly.html) book. You can find it
on Amazon [here](http://www.amazon.com/Assembly-Language-Step---Step-Programming/dp/0470497025/ref=sr_1_1?ie=UTF8&qid=1396917130&sr=8-1&keywords=assembly+language+step+by+step).

Finally, [Mr. Ryan A. Chapman's x86_64 Linux System Call Table](http://blog.rchapman.org/post/36801038863/linux-system-call-table-for-x86-64)
is a good reference.

### Getting Started
Before making nifty assembly programs, you will need an assembler. An assembler is a program that assembles or translates your source
code into machine code. We will use the [Netwide Assembler (NASM)](http://www.nasm.us/) for this purpose. You can install `nasm` via
`apt-get` or `yum`.

You will also need a text editor so you can write your code. Any editor will suffice so long as it saves files in plain text format
without special formatting. In other words: do not use something like OpenOffice Writer. Use `nano` or `vim` instead. ;)

### Why Learn Assembly?
I know what some of you are probably thinking... "Isn't assembly too old to be useful, dude?" On the contrary, my friend. There
are a number of reasons to learn assembly. I've listed some fine examples below.

* Learn how things work on a low level (work with the CPU directly and manage stacks)
* Analyze malware
* Write efficient programs for embedded systems
* Write boot loaders, operating system kernels, and device drivers
* Reverse engineer software to patch it or re-create an application in a higher-level language
* Write performant code for parts of a program that require speed
* It's fun!

Pretty compelling, huh? Not so fast though...

### What Can't You Do?
You must realize a few things about assembly before moving forward. Due to calling conventions and variances with system calls between
operating systems, you cannot assemble the same program you wrote for a 64-bit Linux system and expect it to execute on a machine
running Windows. FreeBSD and other operating systems actually have the same calling convention as Linux in long mode thankfully.
However, it still can take a considerable amount of work to port a program due to the variation of syscalls.

This is even truer when it comes to other hardware architectures such as ARM, PPC, and SPARC which have their own registers and
instruction sets. As a result, you cannot take your shiny new assembly program and run it on your Android phone since these devices
do not make use of AMD64 or EM64T (Intel's implementation of AMD64) processors.

If it is portability you crave, your princess is in another castle. Use a higher-level language such as C, Java, or Ruby. Another
point to note is that most modern compilers do a darn good job of optimizing code. The optimization may be so good that it will
render any assembly code your wetware could whip up as futile. So don't try it, human.

### Things to Consider
There are actually two dialects of the assembly language for the x86 processor. These are referred to as AT&T and Intel syntax.
There are several popular assemblers. Aside from NASM, there's also GNU's GAS, Microsoft's MASM, and Borland's TASM to name a few.
NASM uses the Intel vernacular as does MASM and TASM while GNU's assembler initially used and primarily uses AT&T syntax. This is
not surprising considering Linux's history with Unix and AT&T. It is worth mentioning that GAS can optionally handle Intel syntax now.

Some of the differences between both dialects are in the table below.

|Feature|AT&T|Intel|
|-------|----|-----|
|Required prefixes|$ for immediate values and % for registers|None|
|Operand order|source, destination (similar to Linux/Unix commands)<br>`mov $16, %rax`|destination, source<br>`mov rax, 16`|
|Addressing memory|()|[]|

Do not worry if you fail to understand some of this stuff. We'll touch on `mov`, `rax`, and friends shortly.

### Registers
You can think of registers as little cubby holes within the CPU. If you've ever taken a course in college on C++, your professor may
have provided a similar analogy for addressing memory. Unfortunately, there is nowhere near as many registers as there are memory
addresses in modern computers. Although, AMD64 has increased the number of registers, so we'll take what we can get. Registers provide
the fastest form of access that a computer has. Let me go ahead and introduce you to the family.

|Registers|Size|
|---------|----|
|AL, AH, BL, BH, CL, CH, DL, DH, R8B, R9B, R10B, R11B, R12B, R13B, R14B, R15B|8 bits or 1 byte|
|AX, BX, CX, DX, R8W, R9W, R10W, R11W, R12W, R13W, R14W, R15W, BP, SP, DI, SI, CS, DS, ES, FS, GS, SS, IP, FLAGS|16 bits or 2 bytes|
|EAX, EBX, ECX, EDX, R8D, R9D, R10D, R11D, R12D, R13D, R14D, R15D, EBP, ESP, EDI, ESI, EIP, EFLAGS|32 bits or 4 bytes|
|RAX, RBX, RCX, RDX, R8, R9, R10, R11, R12, R13, R14, R15, RBP, RSP, RDI, RSI, RIP, RFLAGS|64 bits or 8 bytes|

That was a lot to digest. To simplify matters, the majority of the smaller registers above are actually just the same register trimmed
down. For example, `R8B`, `R8W`, and `R8D` are just smaller portions of the 64-bit register `R8`. The actual registers that exist on an
x86-64 CPU (excluding floating point/MMX/3DNow!/SSE/AVX) are: `RAX`, `RBX`, `RCX`, `RDX`, `R8`, `R9`, `R10`, `R11`, `R12`, `R13`, `R14`,
`R15`, `RBP`, `RSP`, `RDI`, `RSI`, `RIP`, `RFLAGS`, `CS`, `DS`, `ES`, `FS`, `GS`, and `SS`. The segment registers (`CS`, `DS`, `ES`, `FS`,
`GS`, and `SS`) were never extended beyond 16 bits and exist for mostly backward compatibility. Furthermore, the lower 8 bits of `BP`,
`SP`, `DI`, and `SI` can be accessed with `REX` prefixing as `BPL`, `SPL`, `DIL`, and `SIL`. However, I won't be covering that topic in
this tutorial.

The 'L' and 'H' in `AL` and `AH` represent the "high" and "low" areas of register `AX`. `AX`, `BX`, `CX`, and `DX` all have these. When
the x86-64 architecture was created, 8 more general purpose registers (GPRs) were added. They are `R8`-`R15`, respectively. These can
also store more compact values by appending a 'B' for byte, 'W' for word, or 'D' for double (double word). There is also a name for a
64-bit value: the quadword. This means 16 (word length) * 4 (quad) = 64 bits.

There are rules and caveats to using these registers which I'll discuss in a section below. Just try and remember `RAX`, `RBX`, `RCX`,
`RDX`, `R8`-`R15`, `RBP`, `RSP`, `RDI`, and `RSI`. You'll be using those the most. I am not going to cover the MMX/3DNow!, SSE, and
AVX registers and their mnemonics since they are outside the scope of this tutorial (as if all the registers above were not enough!)

### Instruction Set
Now that you are familiar with the registers, let's move on to the AMD64 instruction set. Like the IA32 instruction set before it, the
64-bit instructions consist of mnemonics which correspond to opcodes or low-level instructions. These mnemonics enable us to use friendly
symbols to call instructions opposed to referencing difficult-to-remember numeric IDs. I have created a table below containing some of
the many instructions that we will be using for this tutorial. I am also including other common instructions to pique your interest in
the hopes that you will play around with them on your own.

|Instruction|Description|Example|
|-----------|-----------|-------|
|`ADD`|Adds two values and stores the result in the destination (first operand). The sum below will be '8' and it will be stored in `RAX`.|`mov rax, 3`<br>`mov rbx, 5`<br>`add rax, rbx`|
|`CALL`|Performs an unconditional jump to a line of a code. This is not to be confused with `JMP` (which does not save the callee location) This instruction is paired with `RET`.|call foo_label|
|`DEC`|Decrements a value. We initially copy a value of '5' into `RAX` below. The first `DEC` instruction causes the value in `RAX` to change to '4'. The last `DEC` instructions yields '3'.|`mov rax, 5`<br>`dec rax`<br>`dec rax`|
|`INC`|Increments a value. We initially copy a value of '1' into `RAX` below. The first `INC` instruction results in a value of '2'. The last `INC` operation yields '3'.|`mov rax, 1`<br>`inc rax`<br>`inc rax`|
|`JMP`|Jumps to a location in the code unconditionally.|`jmp foo_label`|
|`JNZ`, `JS`, `JZ`, and friends|Conditional jumps that determine the state of various bits in `RFLAGS`. `JNZ` jumps if not zero, `JZ` jumps if zero, and `JS` jumps if signed for example. In this example, the jump to "foo_label" would occur since 1 - 1 = 0. The zero flag would be triggered in `RFLAGS` as a result of `SUB`.|`mov rax, 1`<br>`sub rax, 1`<br>`test rax, rax`<br>`jz foo_label`|
|`LEA`|Load effective address. This stores a memory location and is relevant to pointers from languages like C and C++. Remember that [] references a memory location in NASM.|`lea rax, [foo]`|
|`MOV`|Copies a value into a register. The following example causes `RAX` to contain '32'.|`mov rax, 32`|
|`POP`|Removes the top value from the stack. In the example below, '5' is pushed onto the stack and then popped off and stored in `RBX`.|`mov rax, 5`<br>`push rax`<br>`pop rbx`|
|`PUSH`|Pushes a value onto the stack. '64' is placed onto the stack after the following operation completes.|`mov rax, 64`<br>`push rax`|
|`RET`|Returns to the original location that called `CALL`. This is paired with `CALL`.|`call foo_label`<br>;execution_resumes_here<br>`mov r10, 1`<br>`foo_label:`<br>`mov rax, 1`<br>`ret`|
|`SUB`|Subtract. It functions like ADD, only in reverse. The result is stored in the destination (first operand). The difference will be '4' in the example below.|`mov rax, 8`<br>`mov rbx, 4`<br>`sub rax, rbx`|
|`SYSCALL`|Call a system procedure in long mode. This instruction replaces the `int 0x80` instruction which interrupted the kernel in 32-bit Linux.|`mov rax, some_call`<br>`syscall`|
|`TEST`|Tests values and toggles bits in `RFLAGS` register appropriately. This instruction is frequently used with conditional jumps. In the example below, the value of `RAX` is tested. If the signed bit in `RFLAGS` is on, then jump to another location in the code.|`test rax, rax`<br>`js foo_label`|
|`XOR`|Perform an exclusive or. This is often useful for resetting a register to zero and is more efficient than `MOV REG, 0`.|`xor rax, rax`|

Hopefully that was not too much to absorb. There are a plethora of other instructions such as `CMP`, `DIV/IDIV`, and `MUL/IMUL`, but I would like to keep this tutorial limited.

### Memory Models and Operating Modes
I would like to briefly touch on memory models and operating modes just so you are in the know. If you do not want the extra information,
feel free to skip ahead as this is mostly historical in nature. There are several memory models that exist. These various models handle
the way that memory is addressed. I am sure many of you remember in recent history when 32-bit operating systems were limited to around
4G of RAM. There was something called PAE or Physical Address Extension that enabled you to address >4G of memory. This is actually one
model called the paged memory model.

There is also the similar segmented model where memory is referred to by a segment and offset. This allowed older x86 architectures like
the 8086 to access larger areas of memory (>640K) much like how PAE increases memory accessibility. Using segment and offset is more
tedious than simply using a single real address.

That brings us to the flat model. With this model, you simply refer to an address in its entirety. This is less of a hassle than dealing
with memory segmentation. Fortunately, 64-bit Linux operates in long mode (which increases the address width for a ton of memory) and
uses the flat addressing method. There is no need for memory segmentation (yet) with such a large address space. The masses are happy
for now...

And finally, aside from long mode, there are the three legacy modes that x86-64 supports: protected, real, and virtual. Without going
into too much detail, protected is the mode that 32-bit Linux, FreeBSD, and Windows ran in. Real mode is what DOS operated in and virtual
is a mode that allowed real mode programs to execute in a protected environment.

### Calling Convention
This section is important, so pay attention! If you are familiar with C, C++, Java, Ruby, and others, you will know that functions/methods
take parameters in a certain order.

```c
void do_something(int foo, float bar, char *baz)  
{  
  /* ... */
}  
```

We can see that `do_something()` takes 3 arguments. In order, they are an integer, a float, and a pointer to char. When programming
in assembly for 64-bit Linux, you need to obey the same rules. Registers are designated to accept values in a specific order when
making system calls. The calls you make look for certain data inside these aforementioned registers. If you fail to feed in the proper
data, you may end up with unexpected inputs and/or outputs. Your program may also segfault.

Now don't let this scare you off, but 64-bit Linux actually has two calling conventions. Thankfully, it's easy to know when to use
either. Whenever you are making system calls, you shove stuff into these registers in this particular order:

```
RAX = system_call_id
RDI = parameter #1
RSI = parameter #2
RDX = parameter #3
R10 = parameter #4
R8 = parameter #5
R9 = parameter #6
```

The return value almost always ends up in `RAX`. You do not have to worry about system calls accepting more than 6 parameters.
Otherwise, use the following SysV ABI convention (which happens to be the same for FreeBSD, OS X, and Solaris on x64):

```
RDI
RSI
RDX
RCX
R8
R9
XMM0-7 (these are the SSE registers that accept floating-point values)
```

If you run out of registers to use for passing values to functions in this convention, then you will use the stack which we'll cover
soon. The two conventions are similar with the exception of the floating-point registers and `RCX`/`R10` differ. Microsoft uses a
different calling convention in x64 land (fun fact!)

### Segments
Not to be confused with segmented memory model; segments are simply sections of code. The three you will commonly find throughout your
assembly adventures are the `BSS`, `data`, and `text` segments. The `BSS` or Block Started by Symbol (also jokingly referred to as
"Better Save Space") section is an area where you specify uninitialized variables. For example, I could allocate space for a struct
whose members could be filled by a callee as we will see in part II of this tutorial.

Next comes the `data` segment. This is the area where you define initialized data. One fine example could be a string constant such as
a program name and version. Last but not least, is the `text` segment. This section usually makes up the bulk of the code and is
where you perform instructions.

### The Stack
The stack is simply a construct in memory that can hold data for us. It is common for this data structure to be likened to a stack of
spring-loaded plates at a buffet. Let's say there are no plates at first. We have an empty stack. An employee comes along and begins
adding plates to the stack. This is just like what the `PUSH` instruction would be doing. You `PUSH` a value onto the stack. As the
employee adds more plates, the first plate that was added gets buried deeper and deeper. The same occurs as you `PUSH` more data onto
the stack. Now, some hungry fellow comes along and removes a plate. You guessed it... `POP`! You can either opt to keep or throw away
upper data in the stack to get at your first "plate".

Stack management is important. Many programming languages make use of a stack to hold variables local to a function. It is also a
common source for trouble such as stack smashing attacks/buffer overflows. You can store whatever you'd like in it. The stack is also
used when you perform the `CALL` instruction. The return address is pushed onto the stack and then popped off when you use `RET`. This
happens automagically.

Let's illustrate to help solidify your understanding:

```asm
mov rax, 128
push rax     ; the stack now has '128' on it
mov rax, 256
push rax     ; now the stack has '256' at the top and '128' buried under that
pop rax      ; '256' was removed off the stack and placed into RAX
pop rax      ; RAX now contains '128' and '256' is gone forever!
```

It's not so bad, eh?

### File Formats
This is merely a morsel about file formats. There are a fair amount of them in existence. a.out (old Unix), COFF (Windows PE),
ELF (FreeBSD, HP-UX, Linux, and Solaris), and Mach-O (OS X) are prominent formats. We will be using ELF64 for the programs we create.
This can be done with NASM using the -f (format) option: `nasm -f elf64 -o <output_object_file>.o <input_source_file>.asm`

Linking is also required to make the executable. We can use `ld` for that. The syntax is: `ld -o <executable_name>  <object_file>.o`

### Our First Program
This is the moment you have been waiting for. Look no further. Our first 64-bit assembly program for Linux is below.

```asm
[BITS 64]

section .bss
  ; nada

section .data
  prompt:          db 'What is your name? '
  prompt_length:   equ $-prompt
  name:            times 30 db 0
  name_length:     equ $-name
  greeting:        db 'Greetings, '
  greeting_length: equ $-greeting

section .text
global _start
_start:
  ; Prompt the user for their name
  mov rax, 1                ; '1' is the ID for the sys_write call
  mov rdi, 1                ; '1' is the file descriptor for stdout and our first argument
  mov rsi, prompt           ; prompt string is the second argument
  mov rdx, prompt_length    ; prompt string length is the third argument
  syscall                   ; make the call with all arguments passed

  ; Get the user's name
  mov rax, 0                ; 0 = sys_read
  mov rdi, 0                ; 0 = stdin
  mov rsi, name
  mov rdx, name_length
  syscall
  push rax                  ; store return value (size of name) on the stack... we'll need this for later

  ; Print our greeting string
  mov rax, 1                ; 1 = sys_write
  mov rdi, 1                ; 1 = stdout
  mov rsi, greeting
  mov rdx, greeting_length
  syscall

  ; Print the user's name
  mov rax, 1                ; 1 = sys_write
  mov rdi, 1                ; 1 = stdout
  mov rsi, name
  pop rdx                   ; length previously returned by sys_read and stored on the stack
  syscall

  ; Exit the program normally
  mov rax, 60               ; 60 = sys_exit
  xor rdi, rdi              ; return code 0
  syscall
```

All those lines to do something trivial makes you appreciate writing in higher-level languages, doesn't it? ;)

Anyway, let's assemble and link this bad boy...

```bash
ldilley@develop:~/asm$ nasm -f elf64 -o greet.o greet.asm
ldilley@develop:~/asm$ ld -o greet greet.o
ldilley@develop:~/asm$ ./greet
What is your name? Lloyd
Greetings, Lloyd
```

Try it yourself!

### Breaking it Down
You should understand the segments, instructions, registers, calling convention, and stack. The comments should also help, but let's
discuss the stuff you haven't seen yet...

`[BITS 64]` at the top of our source code specifies the target processor mode. Per the NASM documentation, you do not always need this,
but it certainly helps document the type of code you are writing. A semicolon allows for a single-line comment. The assembler will
ignore any text that comes after it. The "section" keyword denotes the segment. Segments begin with a period in NASM. We have `.bss`,
`.data`, and `.text` segments in our code. Variables and constants are defined by labels. The syntax for a label should be a meaningful
name followed by a colon. Labels are also used as jump points. You can `JMP` and `CALL` to them.

"`db`" stands for "data byte". There is also `dw` (data word), `dd` (data double(word)), `dq` (data quad(word)). This specifies the
storage requirements of the data. In the `BSS` segment, there are similar keywords to reserve data of the same types. Those keywords
are: `resb` (reserve byte), `resw` (reserve word), `resd` (reserve double(word)), and `resq` (reserve quad(word)).

"`equ $-`" is a slick way to calculate the length of a string. "`times 30 db 0`" sets aside 30 bytes of storage space for a string and
zeroes it out. A word of caution: input length is not checked. A string over 30 bytes will be interpreted by the shell. To remedy
this bug, you can either increase the buffer size (change 30 to 255 bytes for example) or sanitize the string/flush anything beyond
the length. I am not including that functionality since it would increase code complexity for the reader. If you _really_ want to
prevent spillage, see [Gunner's NASM I/O Tutorial](http://www.dreamincode.net/forums/topic/286248-nasm-linux-terminal-inputoutput-wint-80h/).
The code is in 32-bit mode, but can be easily adapted by replacing the interrupt instruction with `SYSCALL` and modifying the registers
to meet the 64-bit Linux calling convention.

Finally, "`global _start`" with a label of "`_start:`" signifies the main entry point of the program much like `main()` for C, C++,
and Java.

### Where Do I Go from Here?
I highly recommend using [Ryan Chapman's x86_64 Linux System Call Table](http://blog.rchapman.org/post/36801038863/linux-system-call-table-for-x86-64)
to play around with various system calls. Practice loading the appropriate registers with the proper data. Be aware of what the system
calls are doing however. You do not want to inadvertently overwrite or remove files. I also recommend using the various instructions
to see how they can affect data stored within registers. Try incrementing and decrementing values, perform some jumps, and push/pop
some stuff onto/off the stack. Feel free to use my code as a starting point or the template below.

```asm
[BITS 64]

section .bss

section .data

section .text
global _start
_start:
  ; Your code here

  ; Exit the program normally
  mov rax, 60      ; 60 = sys_exit
  xor rdi, rdi     ; return code 0
  syscall
```

I will also be releasing part II of this tutorial in a week or two. I will discuss endianness, demonstrate analogous C code, and we
will create a 64-bit assembly program for Linux that listens on a TCP port and accepts network connections. How cool is that?

I hope you have enjoyed this tutorial and that it has helped you in some way. Feel free to link to it. If you have any questions or
suggestions, do not hesitate to comment.

### Author and Copyright Information
Copyright © 2014 Lloyd Dilley (originally written on 4/12/2014) under the terms of the [GNU Free Documentation License v1.3](https://www.gnu.org/licenses/fdl-1.3.txt)
