---
layout: post
title:  "Intro to radare2"
subtitle: "Tools"
date:   2017-10-31 00:00:00 -0500
author: "coastal"
header-img: "images/site-resources/radare2-intro-header.png"
---

I'm working on a challenge binary that requires some reverse engineering and it was suggested to work through it with `radare2`. The `radare2` framework (`r2` for short) is an oft-talked about open-source reverse engineering framework that has a reputation for being less than welcoming to new users...

<img src="{{ site.baseurl }}/images/notes-tips-tricks/r2-learning-curve.png">

However, I'm up to the challenge, and got to cracking. I decided to work with the IOLI-crackme challenge, mainly because I found this great [tutorial](https://dustri.org/b/defeating-ioli-with-radare2.html) that uses both `r2` and these binaries, which will help if I hit a real roadblock.

### setup

Installing `r2` couldn't be more straightforward

```
git clone https://github.com/radare/radare2
cd radare2/sys
./install.sh
```

I tried installing via `homebrew`, but was getting some weird errors that tracked back to a dependency. After uninstalling and installing in the recommended fashion, all those issues disappeared.

### 0x00

First thing to do is download the binaries and load the first challenge up for inspection:

```
wget http://pof.eslack.org/tmp/IOLI-crackme.tar.gz
tar -xzvf
cd IOLI-crackme/bin-linux
```

```
coastal@ubuntu32server:~/intro-to-r2/IOLI-crackme/bin-linux$ radare2 crackme0x00
 -- Select your architecture with: 'e asm.arch=<arch>' or r2 -a from the shell
[0x08048360]> aaa
[x] Analyze all flags starting with sym. and entry0 (aa)
[x] Analyze len bytes of instructions for references (aar)
[x] Analyze function calls (aac)
[x] Use -AA or aaaa to perform additional experimental analysis.
[x] Constructing a function name for fcn.* and sym.func.* functions (aan)
```

The `aaa` command is the newer version of the `aa` command which translates to english as 'analyze all'. This command abbreviation is a theme that we see throughout `r2`. As we can see in the output, it is doing a number of analyses on the binary and saving the analyzed information for future use. One of the first things to do when reverse engineering any binary is to check out the strings to get an idea of what the function might be doing. We can do this with the `iz` command.

```
[0x08048360]> iz
vaddr=0x08048568 paddr=0x00000568 ordinal=000 sz=25 len=24 section=.rodata type=ascii string=IOLI Crackme Level 0x00\n
vaddr=0x08048581 paddr=0x00000581 ordinal=001 sz=11 len=10 section=.rodata type=ascii string=Password: 
vaddr=0x0804858f paddr=0x0000058f ordinal=002 sz=7 len=6 section=.rodata type=ascii string=250382
vaddr=0x08048596 paddr=0x00000596 ordinal=003 sz=19 len=18 section=.rodata type=ascii string=Invalid Password!\n
vaddr=0x080485a9 paddr=0x000005a9 ordinal=004 sz=16 len=15 section=.rodata type=ascii string=Password OK :)\n
```

In our challenge file, it appears to take in a password and compare it to an expected password. Let's run the file and see if that is the case

```
coastal@ubuntu32server:~/intro-to-r2/IOLI-crackme/bin-linux$ ./crackme0x00
IOLI Crackme Level 0x00
Password: password
Invalid Password!
```

That does in fact seem to be what is going on. Now given that this is the first challenge (and likely fairly easy), we notice that one of the strings that `r2` found looks suspiciously like a password: `250382`. What happens if we enter that?

```
coastal@ubuntu32server:~/intro-to-r2/IOLI-crackme/bin-linux$ ./crackme0x00
IOLI Crackme Level 0x00
Password: 250382
Password OK :)
```

Pretty easy :)

However, if we hadn't noticed that nice little obvious answer, we could dive into the disassembly. First thing to do would be to check out the main function, which we can do with `pdf@sym.main`, or `print` `disassembled` `function` @ `offset`.

```
[0x0804a8a0]> pdf@sym.main
            ;-- main:
/ (fcn) main 127
|   main ();
|           ; var int local_18h @ ebp-0x18
|           ; var int local_4h @ esp+0x4
|              ; DATA XREF from 0x08048377 (entry0)
|           0x08048414      55             push ebp
|           0x08048415      89e5           mov ebp, esp
|           0x08048417      83ec28         sub esp, 0x28               ; '('
|           0x0804841a      83e4f0         and esp, 0xfffffff0
|           0x0804841d      b800000000     mov eax, 0
|           0x08048422      83c00f         add eax, 0xf
|           0x08048425      83c00f         add eax, 0xf
|           0x08048428      c1e804         shr eax, 4
|           0x0804842b      c1e004         shl eax, 4
|           0x0804842e      29c4           sub esp, eax
|           0x08048430      c70424688504.  mov dword [esp], str.IOLI_Crackme_Level_0x00_n ; [0x8048568:4]=0x494c4f49 ; "IOLI Crackme Level 0x00\n"
|           0x08048437      e804ffffff     call sym.imp.printf         ; int printf(const char *format)
|           0x0804843c      c70424818504.  mov dword [esp], str.Password: ; [0x8048581:4]=0x73736150 ; "Password: "
|           0x08048443      e8f8feffff     call sym.imp.printf         ; int printf(const char *format)
|           0x08048448      8d45e8         lea eax, [local_18h]
|           0x0804844b      89442404       mov dword [local_4h], eax
|           0x0804844f      c704248c8504.  mov dword [esp], 0x804858c  ; [0x804858c:4]=0x32007325
|           0x08048456      e8d5feffff     call sym.imp.scanf          ; int scanf(const char *format)
|           0x0804845b      8d45e8         lea eax, [local_18h]
|           0x0804845e      c74424048f85.  mov dword [local_4h], str.250382 ; [0x804858f:4]=0x33303532 ; "250382"
|           0x08048466      890424         mov dword [esp], eax
|           0x08048469      e8e2feffff     call sym.imp.strcmp         ; int strcmp(const char *s1, const char *s2)
|           0x0804846e      85c0           test eax, eax
|       ,=< 0x08048470      740e           je 0x8048480
|       |   0x08048472      c70424968504.  mov dword [esp], str.Invalid_Password__n ; [0x8048596:4]=0x61766e49 ; "Invalid Password!\n"
|       |   0x08048479      e8c2feffff     call sym.imp.printf         ; int printf(const char *format)
|      ,==< 0x0804847e      eb0c           jmp 0x804848c
|      ||      ; JMP XREF from 0x08048470 (main)
|      |`-> 0x08048480      c70424a98504.  mov dword [esp], str.Password_OK_:__n ; [0x80485a9:4]=0x73736150 ; "Password OK :)\n"
|      |    0x08048487      e8b4feffff     call sym.imp.printf         ; int printf(const char *format)
|      |       ; JMP XREF from 0x0804847e (main)
|      `--> 0x0804848c      b800000000     mov eax, 0
|           0x08048491      c9             leave
\           0x08048492      c3             ret

```


Looking at `0x08048469` we see a `strcmp` call and a jump to `printf` of 'Password OK :)\n' if the compared strings are indeed equal. If we look at the values that were compared, one of them is our previous string of interest:

```
0x0804845e      c74424048f85.  mov dword [local_4h], str.250382 ; [0x804858f:4]=0x33303532 ; "250382"
```

The other is our input from `scanf`. It becomes clear that our input needs to be `250382`. And of course...

```
coastal@ubuntu32server:~/intro-to-r2/IOLI-crackme/bin-linux$ ./crackme0x00
IOLI Crackme Level 0x00
Password: 250382
Password OK :)
```

I will likely add to this post as I progress instead of making more posts just to keep everything in one place. `0x01` up next.















