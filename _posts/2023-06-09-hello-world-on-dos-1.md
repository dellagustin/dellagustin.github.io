---
layout: post
title:  "Saying Hello in the PC-DOS 1.0 Operating System"
date: 2023-06-26
categories: [retro-computers]
tags: ibm-pc, pc-dos
---

I like spending some of my time learning more about the architecture of retro computers. That's an interest that comes from a very early love for computers in my life and that includes video games. From time to time I read about this topics and some time ago I was interested in the IBM PC (IBM 5150), launched in 1981.

That's a very important device in the history modern computers, as it gave origin to the x86 architecture and also the MS-DOS/PC-DOS (Disk Operating System) family of operating systems. Note that DOS is a term that can refer to other operating systems of the time, but throughout this post, DOS will refer to MS-DOS/PC-DOS.

After seeing a video of someone writing assembly code on DOS using the `DEBUG.COM` application, I thought it would be a nice little exercise, really, just a classic _"Hello world"_, but in a very old school way.

In this post I'll talk briefly about the architecture of these early machines and show you how to write a _Hello world_ in machine code for the IBM PC in DOS 1.0.

## The IBM PC architecture in a nutshell

Understanding the system architecture in depth is not really needed to write a _"Hello World"_, but the whole point of writing it in assembly (actually direct machine code) is to do it as close as possible to the hardware and experience some of the nitty gritty of these architectures.

In general the architecture of 8-bit microcomputers that era would look like in the diagram below:

![General 8-bit microcomputer architecture](/assets/retro-computers/general-8-bit-architecture.svg)

Specific to our scenario:

- the CPU is an Intel 8088,
- the RAM ranges form 16 to 640 KB,
- the Video will be a Monochrome Display Adapter (MDA), although this does not specifically affect our task, and
- the Read and Write Storage will be a 360 KB floppy disk, from where the operating system is load, and where we will store our program.

Looking into the software architecture, this is how it looks like:

![Hardware and software layers](/assets/retro-computers/hardware-and-software-layers.drawio.svg)

At start up, the CPU will fetch instructions directly from the BIOS.
The BIOS, Basic Input and Output System, will load the Operating System (OS) and provide an abstraction layer for it so that it can interface with the hardware. The OS, in turn, will provide some higher level features directly to the user, like interacting with file systems and loading and running programs, as well as providing some basic reusable routines and serving as yet another abstraction layer for the programs it loads and runs.

## Now, to the action

I don't have IBM PC or DOS 1.0 disk at hand, and I assume you don't have one either.

Luckily, it is very easy to emulate this device today. I'll be using the excellent site PC pcjs.org ([here](https://www.pcjs.org/machines/pcx86/ibm/5150/mda/) is a direct link to the IBM PC emulator), which emulates devices directly in the browser.

PCem is also a good option to emulate it on your PC.

Before we go on, let's check what our program is going to do.
We need to do basically two things:

1. Tell the operating system to show a message on the screen,
2. Tell the operating system we are done.

We are going to do that with the DOS API. To consume this API we need to invoke software interrupts with the `INT` instruction. That will divert the flow of the code being executed by the processor to code that belongs to the operating system, which will bring the flow back to our program once it is ready (or not, if we are done).

The main DOS API uses interrupt vector `21h`, and the function to display the message is selected by setting the register `AH` to `09h` and the `DX` register to the memory address where the message is located. This function considers that messages end in the `$` character. To end the program, we call the interrupt after setting register `AH` to `00h`.

Now, some nitty gritty about the `DEBUG` version in DOS 1.0:

- it does not assemble (unlike the avengers), so we need to "code" using machine code directly - later versions can create machine code from assembly commands,
- it does not save your program to disk, unless the file already exists.

So, first, lets create the empty file `HELLO.COM` with the following command:

```
copy con hello.com
```

then press _CTRL+Z_ (you will see `^Z` on the screen) and press _ENTER_, you will see the message `1 File(s) copied`.

Let's get to the fun part. We'll start using `DEBUG`, type the following command:

```
debug hello.com
```

You will be greeted by a simple `-` followed by a blinking cursor.

We will add the following instructions in machine code, starting from the memory address `100h`:

```
Memory address (in hex)   Instruction   Machine code (in hex)
100                       MOV AH,09     B4 09
102                       MOV DX,010B   BA 0B 01
105                       INT 21        CD 21
107                       MOV AH,00     B4 00
109                       INT 21        CD 21
```

I have converted those instructions to machine code manually using this opcodes list: <https://www.pastraiser.com/cpu/i8088/i8088_opcodes.html>

Notice that on memory address `102h` we are setting the register `DX` to `10Bh`, it is exactly the address that comes after our last instruction, we will place our message there later.

To enter those instructions, use the following command (remember we are still inside `DEBUG`):

```
e 100 B4 09 BA 0B 01 CD 21 B4 00 CD 21
```

Now check if you did it correctly with the `u 100` (unassemble) command. `DEBUG` will print the instructions and you can compare it with the ones above.

Great! Now, let's enter the message! use the command below:

```
e 10B "Hello world!$"
```

All right, we are ready to test! Use the following command:

```
g=100
```

With this, debug will run the code you have written, starting on the memory address `100h`, you should see the message _Hello world! Program terminated normally_.

All there is left to do now is to save our program to disk. For that, we need to write the program size to the register `CX`. Use the command `RCX` and enter the value `18` (it is the size of our program in bytes, 24, converted to hex), then use the command `w`. This will write your program to disk. Now quit debug with the command `q` and run your program by typing its name in the command prompt!

```
hello
```

Did you get your message on the screen? Thrilling, isn't it??

I hope you enjoyed this retro "Hello world" experience!

## Bonus - more on the PC-DOS / MS-DOS history

If you would like to learn more about the history behind PC-DOS and MS-DOS, I recommend the following videos:

- [Why DOS Was (and Is) a Thing](https://youtu.be/3E5Hog5OnIM) - YouTube - 32 minutes long
- [What is IBM PC DOS 2000? - History and Unboxing](https://youtu.be/dHR3xoRT-4M) - YouTube - 26 minutes long

## References

- https://en.wikipedia.org/wiki/IBM_Personal_Computer
- https://en.wikipedia.org/wiki/Intel_8088
- https://en.wikipedia.org/wiki/IBM_PC_DOS
- https://en.wikipedia.org/wiki/DOS_API
- https://thestarman.pcministry.com/asm/debug/debug.htm
- Emulators, disk images and roms
    - https://www.pcjs.org/machines/pcx86/ibm/5150/mda/
    - https://www.pcem-emulator.co.uk/downloads.html
    - https://github.com/BaRRaKudaRain/PCem-ROMs/tree/master/ibmpc
