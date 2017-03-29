---
title:  "Microcorruption -- New Orleans (Level 1)"
categories:
  - Microcorruption
tags:
  - challenge
  - reverse
  - msp430
  - new orleans
  - assembly
  - microcorruption
  - writeup
---
<div class="notice--info">
<strong>Useful info:</strong>
<ul>
<li>Microcorruption tutorial</li>
<li><a href="https://microcorruption.com/manual.pdf">Lock Manual</a></li>
<li><a href="http://www.ti.com/lit/ug/slau144j/slau144j.pdf">MSP430x2xxx Family User's Guide (Chapter 3)</a></li>
</ul>
</div>

This first challenge is quite simple and can be solved with at least 2 methods. The first method is good to get some pratice with the debugger, while the second one just requires static analysis.

# Static analysis

In these challenges we're given the assembly code, so we'll start by taking a brief look at it. In this case, the code is quite small so there's not much to it.

Let's start by looking at the program entry point, in this case the `main` routine:

```
4438 <main>
4438: add   #0xff9c, sp
443c: call  #0x447e <create_password>
4440: mov   #0x44e4 "Enter the password to continue", r15
4444: call  #0x4594 <puts>
4448: mov   sp, r15
444a: call  #0x44b2 <get_password>
444e: mov   sp, r15
4450: call  #0x44bc <check_password>
4454: tst   r15
4456: jnz   #0x4462 <main+0x2a>
4458: mov   #0x4503 "Invalid password; try again.", r15
445c: call  #0x4594 <puts>
4460: jmp   #0x446e <main+0x36>
4462: mov   #0x4520 "Access Granted!", r15
4466: call  #0x4594 <puts>
446a: call  #0x44d6 <unlock_door>
446e: clr   r15
4470: add   #0x64, sp
4474 <__stop_progExec__>
4474: bis   #0xf0, sr
4478: jmp   #0x4474 <__stop_progExec__+0x0>
```

We can understand the program flow just by looking at the strings. The user is asked for a password and if it matches the correct password, the "Access Granted!" string is printed and the `unlock_door` routine is called, which is the objective for these challenges, if the input doesn't match we get the "Invalid password; try again." string.

To test our assumption of the program flow, we can run the code in the debugger, enter a random password and verify we get the "Invalid password; try again." string.

# Challenge Solving

Having a basic idea on what the program does and what we want it to do, we can go look into solving it.

## Method \#1

To reach `0x446a` and unlock the door, the `check_password` routine must return `true`, so that the jump at `0x4456` is taken. Let's look at the routine:

```
    44bc <check_password>
    44bc: clr   r14
  ╭─44be: mov   r15, r13
  ↑ 44c0: add   r14, r13
  ↑ 44c2: cmp.b @r13, 0x2400(r14)
╭─↑─44c6: jne   #0x44d2 <check_password+0x16>
↓ ↑ 44c8: inc   r14
↓ ↑ 44ca: cmp   #0x8, r14
↓ ╰─44cc: jne   #0x44be <check_password+0x2>
↓   44ce: mov   #0x1, r15
↓   44d0: ret
╰───44d2: clr   r15
    44d4: ret
```

This routine implements a loop that compares the user input with the correct password, we can see it compares the password byte by byte (`cmp.b @r13, 0x2400(r14)`) and that the correct password is 7+1 (null byte) bytes long (`cmp #0x8, r14`). Furthermore, we can see the correct password is stored at `0x2400`.

With this information we can run the debugger and retrieve the password from memory with the read command: `r 2400 8`.

## Method \#2

Taking another look at the `main` routine, we see an interesting routine at the start named `create_password`:

```
447e <create_password>
447e:  3f40 0024      mov	#0x2400, r15
4482:  ff40 7700 0000 mov.b	#0x77, 0x0(r15)
4488:  ff40 4800 0100 mov.b	#0x48, 0x1(r15)
448e:  ff40 2a00 0200 mov.b	#0x2a, 0x2(r15)
4494:  ff40 5e00 0300 mov.b	#0x5e, 0x3(r15)
449a:  ff40 3400 0400 mov.b	#0x34, 0x4(r15)
44a0:  ff40 7d00 0500 mov.b	#0x7d, 0x5(r15)
44a6:  ff40 6300 0600 mov.b	#0x63, 0x6(r15)
44ac:  cf43 0700      mov.b	#0x0, 0x7(r15)
44b0:  3041           ret
```

This is a pretty simple routine that sets the password byte by byte. We can just submit the 7 bytes (checking the input is hex encoded) or decode and send them.

## Solution

Here's the SHA1 of the solution:

93170542357ce5134027fb9e51ade8a6f5376124
