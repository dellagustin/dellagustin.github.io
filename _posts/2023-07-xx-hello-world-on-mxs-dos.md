---
layout: post
title:  "Saying 'Hello World' in the MSX-DOS Operating System"
date: 2023-07-02
categories: [retro-computers]
tags: msx, msx-dos
---

I recently published a blog post on [Saying 'Hello World' in the IBM-PC with MS-DOS 1.0](./2023-06-09-hello-world-on-dos-1.md). After that, I got curious about doing the same on MSX-DOS.

MSX-DOS was a port of MS-DOS/PC-DOS for the MSX computers on 1984. The MSX was not a single computer from a specific company, but rather a standardized architecture for 8-bit machines established by a partnership of Microsoft and ASCII Corporation in Japan.

The first computer based on this standard was the Mitsubishi ML-8000, released in 1983.

## Saying Hello

This time I'll skip a lengthier architecture talk, as I already touched that in my previous post. What we need to know:

- The MSX computers that a Zilog Z80 (or compatible) under the hood
- Like in MS-DOS, there was an underlying API for basic stuff, like showing a string
- Like in MS-DOS, "external commands" are `.COM` files with pure machine code that are loaded on the memory address `0x100` and executed from there 
- Unlike MS-DOS, the MSX-DOS API does not use software interrupts, but rather a simple subroutine `CALL`

Now, how do we create a `HELLO.COM` command uses the MSX-DOS API to print a _Hello World_ message in the screen?
Well, MS-DOS had the handy `DEBUG` application, which allowed us to write machine code to memory, executed it, unassemble it, inspect it and save it to disc, but that's not present in MSX-DOS (even though a good soul wrote a port for it, see <https://gitlab.com/Emily82/debugger-debug.com-for-msx>).

So, reading the MSX2 Technical Handbook, Chapter 3 - MSX-DOS, Section 2.4 - _External Commands_, I got an interesting surprise. It suggests the following BASIC program for writing bytes to a file:

```basic
100 '***** This program makes "CLS.COM" *****
110 '
120 OPEN "CLS.COM" FOR OUTPUT AS #1
130 '
140 FOR I=1 TO 8
150   READ D$
160   PRINT #1,CHR$(VAL("&H"+D$));
170 NEXT
180 '
190 DATA 1E,0C,0E,02,CD,05,00,C9
```

Isn't that mind blowing? Writing a basic program to create a MSX-DOS program (a.k.a. external command).

So, we can adapt that to create our own `HELLO.COM` command, all we need is to figure out the bytes we need for the instructions that will call the MSX-DOS API to print our message in the screen.

Let's look into the API then, referring again to the Technical Handbook, now section 4 of Chapter 3, _System Call Usage_.

It says we need to enter the API function number in the C register, and then call the sub-routine at address `0x0005`. The function to send a string to the screen is `0x09`, and it takes a single argument, the memory address where the string is. Just like in MS-DOS, the character `$` indicates the end of the string.

Let's write the code to do that in assembly, just as a reference, because we will need to translate it into machine language:

```
LD C,09H
LD DE,0109H 
CALL 0005H
RET 
```

Note that the second instruction is setting the register DE to `0x109`, this is the address where the string will be. I'm already setting it here, but it is only because I already translated this to machine code, and I know exactly the memory address that follows the `RET` instruction. Keep in mind, this code will be loaded into the memory address `0x100`.

## References

- https://en.wikipedia.org/wiki/MSX
- https://en.wikipedia.org/wiki/MSX-DOS
- https://konamiman.github.io/MSX2-Technical-Handbook/
