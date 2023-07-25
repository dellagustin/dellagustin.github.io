---
layout: post
title:  "Saying 'Hello World' in the MSX-DOS Operating System"
date: 2023-07-25
categories: [retro-computers]
tags: msx, msx-dos
---

I recently published a blog post on [Saying 'Hello World' in the IBM-PC with MS-DOS 1.0](./2023-06-09-hello-world-on-dos-1.md). After that, I got curious about doing the same on MSX-DOS.

MSX-DOS was a port of MS-DOS/PC-DOS for the MSX computers on 1984. The MSX was not a single computer from a specific company, but rather a standardized architecture for 8-bit machines established through a partnership of Microsoft and ASCII Corporation in Japan.

The first computer based on this standard was the Mitsubishi ML-8000, released in 1983.

## Saying Hello

This time I'll skip a lengthier architecture talk, as I already touched that in my previous post. What we need to know:

- The MSX computers have a Zilog Z80 (or compatible) as its CPU under the hood
- Like in MS-DOS, there was an underlying API for basic stuff, like showing a string
- Like in MS-DOS, "external commands" are `.COM` files with pure machine code that are loaded on the memory address `0x100` and executed from there 
- Unlike MS-DOS, the MSX-DOS API does not use software interrupts, but rather a simple subroutine invoked with the  `CALL` instruction, moving the execution to the area in RAM where the Operational System is loaded

The Z80 CPU was used in many other computers of this era and also arcade games, such as the iconic Pac-Man.

Now, how do we create a `HELLO.COM` command using the MSX-DOS API to print a _Hello World_ message in the screen Well, MS-DOS had the handy `DEBUG` application, which allowed us to write machine code to memory, executed it, unassemble it, inspect it and save it to disc, but that's not present in MSX-DOS (even though a good soul wrote a port for it, see <https://gitlab.com/Emily82/debugger-debug.com-for-msx>).

So, reading the [MSX2 Technical Handbook, Chapter 3 - MSX-DOS](https://konamiman.github.io/MSX2-Technical-Handbook/md/Chapter3.html), Section 2.4 - _External Commands_, I got an interesting surprise. It suggests the following BASIC program for writing bytes to a file:

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

This is the very simple external command `CLS.COM`, written byte-by-byte to the disk.

Isn't that mind blowing? Writing a basic program to create a MSX-DOS program. There are of course more sophisticated ways of doing that, but I wanted to do it with the tooling I had just with the computer plus the operating system, no extra tooling.

We can adapt that to create our own `HELLO.COM` command, all we need to do is to figure out the bytes that we need for the instructions that will call the MSX-DOS API to print our message in the screen, and also the message itself.

Let's look into the API then, referring again to the Technical Handbook, now section 4 of Chapter 3, _System Call Usage_.

It says that we need to enter the API function number in the C register of the CPU, and then call the sub-routine at address `0x0005`. The function number to send a string to the screen is `0x09`, and it takes a single argument, the memory address where the string is located. Just like in MS-DOS, the character `$` indicates the end of the string.

Let's write the code to do that in assembly, just as a reference, because we will need to translate it into machine language:

```
LD C,09H
LD DE,0109H 
CALL 0005H
RET 
```

Note that the second instruction is setting the register DE to `0x109`, this is the address where the string will be. I'm already setting it here, but it is only because I already translated this to machine code, and I know exactly the memory address that follows the `RET` instruction. Keep in mind, this code will be loaded into the memory address `0x100`.

Ok, now let's translate that assembly code to machine code, using this convenient [Z80 opcode table](https://clrhome.org/table). The result is (memory address and opcode in hex):

```
M    OP        ASM
100  0E 09     LD C,09H
102  11 09 01  LD DE,0109H
105  CD 05 00  CALL 0005H
108  C9        RET 
```

So, the code itself is the following sequence of bytes: 0E, 09, 11, 09, 01, CD, 05, 00, C9

We still need to append our text message to that (`Hello World!$`). To convert that to hex, I'll use the list of MSX characters and control codes at <https://www.msx.org/wiki/MSX_Characters_and_Control_Codes>.

The result is:  48, 65, 6c, 6c, 6f, 20, 57, 6f, 72, 6c, 64, 21, 24

Now, all we need to do is adapt the basic program:

```basic
100 '***** This program makes "HELLO.COM" *****
110 '
120 OPEN "HELLO.COM" FOR OUTPUT AS #1
130 '
140 FOR I=1 TO 22
150   READ D$
160   PRINT #1,CHR$(VAL("&H"+D$));
170 NEXT
180 '
190 DATA 0E,09,11,09,01,CD,05,00,C9,48,65,6C,6C,6F,20,57,6F,72,6C,64,21,24
200 CLOSE
```

Note that I added a `CLOSE` statement. It was missing on the example from the manual. Without it, the content was not actually written to the file!

At this point it is good to mention that you can emulate an MSX on the web at <https://webmsx.org/>.

All we need to do is type the program above in the basic interpreter and run it, it will create the file `HELLO.COM`.

Then we go back to the MSX-DOS and execute it, and the magic happens! You will be greeted with a _Hello World!_ message!

In my setup, I loaded the MSX-DOS disk to disk 1 (A), and created the BASIC program and HELLO.COM program on disk 2 (B).

Here are some screenshots:

![Basic code for writing a hello world command in MSX-DOS, webmsx emulator](/assets/retro-computers/WebMSX-DOS-hello-basic.png)

![Hello world command running in MSX-DOS, webmsx emulator](/assets/retro-computers/WebMSX-DOS-hello-cmd.png)

## Closing words

I was quite surprised with the fact that I ended up writing a simple BASIC program to create the external command for me, but at the end this was a fun experience.

The MSX was an interesting spec, but arriving two years later than the IBM-PC, it did not have a big chance of becoming THE standard for home computers (which IBM did without even wanting to do so).

These "hello world"s are an interesting way to start grasping how these machines work on a lower level. I hope you like it and try it for yourself! 

## References

- https://en.wikipedia.org/wiki/MSX
- https://en.wikipedia.org/wiki/MSX-DOS
- https://konamiman.github.io/MSX2-Technical-Handbook/
