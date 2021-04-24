---
layout: post
title:  "My first ROM Hack, fixing the infinite magic on Master System's Ninja Gaiden"
date:   2021-04-24
categories: [videogames, blogging, emulation, rom hacking]
tags: sms rom hack ninja gaiden
---

## TL;DR

I have made my first ROM hack, fixing an issue on Master System's Ninja Gaiden, where get infinite magic points once you collect 999 or more of them.

Here is the BPS patch: [Ninja-Gaiden-Europe-avoid-infinite-magic.bps](/assets/downloads/Ninja-Gaiden-Europe-avoid-infinite-magic.bps)

## A bit of my motivation

Growing up in Brazil, I owned a Sega Master System when I was kid. The SMS was far more available, and probably more popular then the Nintendo Entertainment System, as it was very well distributed by a local company, Tec Toy.

I'm a long time gamer, and a software developer by trade, with some knowledge on electronics and retro computers architecture, more theoretical than practical.

Putting all that together, it is not a surprise that I have a certain fascination for retro games and their inner workings.

For quite some time now I have been wanting to get something done in that area, and ROM hacking is a nice way to get acquainted with the topic.

In my spare time I am also an occasional podcaster. At [Fliperama de Boteco](https://fliperamadeboteco.com/), a Brazilian podcast, we talk mostly about retro games, and some time ago we published an [episode on Ninja Gaiden for the Master System](https://fliperamadeboteco.com/2019/05/30/fliperama-de-boteco-179-ninja-gaiden-master-system/). One of things that got our attention was the fact that once you collected more than 999 magic points, you could use special weapons and magic without running out of points, specially with a fire magic that makes you invincible, and normally cost you 50 magic points. That made the game too easy, and to me it felt like a bug, as you could collect enough magic to do this soon in the game.

Recently, someone posted about this game in a Brazilian Master System group on Facebook, and one of the replies was "I pray that someone will make a hack to fix the bug of the thousand magic points ... it makes the game too easy when it reaches 999". I took that as an interesting challenge to get myself initiated in ROM hacking, and went for it.

## How I did it

The fist step was to find out where in the RAM memory the information about the number of magic points.

I started searching for it using the emulator BizHawk, which contains a _RAM Search_ function.

Using this function, I eventually found out that magic points were stored in the following addresses:

- `0x1FB8` - first digit
- `0x1FB9` - second digit
- `0x1FBA` - third digit
- `0x1FBB` - this one is normally 0, but it is set to 255 (`0xFF` in hexadecimal) when you collect more than 999 points

For example, if you have 150 magic points, these will be the values stored addresses above:

- `0x1FB8` - 0
- `0x1FB9` - 5
- `0x1FBA` - 1
- `0x1FBB` - 0

One important information, this addresses are relative to `0x0000`, but the RAM memory on the Master System is mapped to the address range from `0xC000-0xDFFF` (and mirrored on the address range `0xE000-0xFFFF`), so `0x1FB8` is actually mapped at `0xDFBB` or `0xFFBB`.

Now that I have identified this addresses, I started looking for the code that changed their content.

For that I used a different emulator, [Emulicious](https://emulicious.net/), which has a very good disassembler and debugger.

I added breakpoints to stop the execution where this addresses were read or written, and started doing stuff like using magic and getting items that granted magic points, also reaching over 999.

Ultimately I discovered that `0xDFBB` was used to protect the magic counter from overflowing, but I also found out that it was checked at the beginning of routine that subtracted magic points when magic was used:

![Emulicious debugging, routine at 0x50D9](/assets/ninja-gaiden-emulicious-debugging.png)

```asm
_LABEL_50D9_:
    ld a, (_RAM_DFC3_)
    or a
    jr nz, _LABEL_513E_
    ld a, (_RAM_DFBB_)    ; load content of 0xDFBB into register a
    or a                  ; bitwise OR of a with a, store the result in a
    jr nz, _LABEL_513E_   ; jump to 0x513E if the result of last operation (a) is not zero
    ...                   ; continue to subtraction routine

_LABEL_513E_:
    scf
    ret
```

So I figured that setting the content of `0xDFBB` back to zero instead of checking it and jumping should give me the result I wanted.

To do that, I used the Hex Editor [wxHexEditor](https://sourceforge.net/projects/wxhexeditor/).

First thing was look at the address `0x50D9` (20697 in decimal) and locate the opcodes for the check:

![HexEditor at 0x50D9](/assets/ninja-gaiden-hexeditor-50D9.png)

Now, without calling another routine, I had 6 bytes to do what I wanted, set the content of `0xDFBB` with `0`.

There are more sophisticated ways of doing this, like using a Z80 assembler like WLA-DX, but that would be like bringing a gun to a knife, fight. I did not even had to look for the opcodes of the instructions I wanted to use to replace the check:

```
    ld a, $00
    ld ($DFBB), a
``` 

there was a very similar piece of code just above the one I wanted to change:

![HexEditor at 0x50D0](/assets/ninja-gaiden-hexeditor-50D0.png)

It was only 5 bytes, I just had to adapt so that it was `ld a, $00` instead of `ld a, $FF`, and fill the remaining byte with `0`, which is the instruction `nop` (_No Operation_):

![HexEditor at 0x50D0, hack](/assets/ninja-gaiden-hexeditor-50D9-hack.png)

Now, I tried to load the game, and...

![Patch with wrong checksum](/assets/ninja-gaiden-patch-with-wrong-checksum.png)

Ooops, something is wrong...

The ROMs have a checksum stored at the address `0x7FFC`, which is checked by the BIOS. After hacking the ROM we need to adjust the checksum. To calculate the checksum for the patch I used the command line tool [SMS Check](https://www.smspower.org/Development/SMSCheck):

```
C:\Emuladores\Roms-SMS>smscheck.exe "Ninja Gaiden (Europe) - avoid infinite magic.sms"
SMS Check 0.24 - by Omar Cornut (Bock), Apr 19 2016
http://www.smspower.org
--

[./Ninja Gaiden (Europe) - patched with failure.sms]
Size ...... 262144
Header .... 54 4D 52 20 53 45 47 41 00 00 AD 4C 01 71 00 40
System .... SMS Export (4)
Size ...... 256k+
CheckSum .. 4BB3, Bad. Should be 4CAD
FullSum ... 1674F91
```

All, right, now I adjusted the checksum in the ROM, again using the hex editor:

![HexEditor at 0x7FFA, hack](/assets/ninja-gaiden-hexeditor-7FFA-hack.png)

And, there we go:

![Ninja Gaiden - no infinite magic](/assets/ninja-gaiden-no-infinite-magic.gif)

No more infinite fire magic!

Now I had my hack ready, so it is time to produce a BPS patch, I used a tool called [Floating IPS](https://www.romhacking.net/utilities/1040/).

![Floating IPS](/assets/floating-ips.png)

And there we have it, a patch file ready to be applied: [Ninja-Gaiden-Europe-avoid-infinite-magic.bps](/assets/downloads/Ninja-Gaiden-Europe-avoid-infinite-magic.bps)

## Closing words

It was a fun ride, getting acquainted with Z80 assembly takes a bit of patience, my last contact with assembly was around 20 years ago... managing to read it after sometime is quite satisfactory. I hope this blog post helps other people that would like to get started with ROM hacking.

A huge thanks go to the people maintaining the excellent content on the website smspower.org.

## References
- https://en.wikipedia.org/wiki/Ninja_Gaiden_(Master_System)
- http://www.romhacking.net/forum/index.php?topic=19594.msg276409#msg276409
- https://landley.net/history/mirror/cpm/z80.html
- https://wikiti.brandonw.net/index.php?title=Z80_Instruction_Set
- [Trainers - Do It Yourself](https://www.smspower.org/Articles/TrainersDoItYourself) at smspower.org
- [Z80 Instructions Set](https://www.smspower.org/Development/InstructionSet) at smspower.org
- [ROM Header, Checksum](https://www.smspower.org/Development/ROMHeader#Checksum7ffa2Bytes) at smspower.org
- [How to Use BPS and IPS Patch Files](https://www.smspower.org/Hacks/HowToUseBPSAndIPSPatchFiles) at smspower.org
- [Floating IPS](https://www.romhacking.net/utilities/1040/) at romhacking.net
