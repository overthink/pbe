---
layout: post
title: Microcorruption challenge
published: false
---

Back in January, [Square and
Matasano](http://corner.squareup.com/2014/01/capture-the-flag.html) came up
with an [interesting challenge](https://microcorruption.com) that involves
hacking the firmware of a fictional device running on a Texas Instruments
[MSP430](http://en.wikipedia.org/wiki/MSP430) chip.  The challenge is structred
as a series of levels of increasing difficulty, where you must find and exploit
some vulnerability in the code to get to the next level.

Coming into this I had no clue at all about how low level code and exploits
worked, but the challenge is setup for beginners, so it was a great way to gain
some insight.

Below are my notes and solutions.  Do yourself a favour and actually do the
challenge yourself!  It'll only take... tens of hours!  But it's well worth it.

### Tutorial

Well, they lead you through this one.  In the `check_password` function you can
see it incrementing a count in `r12` after each character of the password is
read.  Then if the counter is equal to nine (meaning eight characters read)
`cmp #0x9, r12`, it signals the password is good (`mov #0x1, r15`) and we're
done.

My answer: `aaaaaaaa` (any eight-char password)

### New Orleans

The first thing done in `main` is a call to `create_password` which promptly
puts the password into main memory at address `0x2400`.  Then you just look at
it.  Doesn't get a lot easier.

My answer: `z99zlHo`

### Sydney

This one just compares the bytes in the entered password to a bunch of
constants in code.  You can read the password right out of `check_password`.
If you're like me and have no idea what you're doing, you'll likely trip up on
endianness here.  The first check in `check_password` is `cmp #0x3024,
0x0(r15)`, meaning the first two bytes are `30` and `24` (hex).  But the MSP430
is little-endian, so the least significant byte (`24`) is stored in the lower
(smaller) memory address.  So you have to enter `2430` (hex) for the first two
bytes.  Same procedure for the next six bytes.

My answer (hex): `2430426b5466706d`

### Hanoi

What happens right before the call to `unlock_door` in the `login` procedure?
Checking that the contents of a fixed address with the constant `0x3` (i.e.
`cmp.b #0x3, &0x2410`).  Working backwards I need to get `0x3` into address
`0x2410`.  Fortunately, you can overflow the input here.  Passwords are stored
starting at `0x2400` and while they're supposedly not to exceed 16 chars in
length, you can enter up to 28.  Why?  They pass `0x1c` (decimal 28) as the
second arg to interrupt `0x02` in `getsn`, meaning it'll read 28 bytes of
input.  Anyway, all we need to do is make sure the 17th byte of the password,
which will be stored at `0x2410` is 0x3.

My answer (hex): `aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa0300`

### Cusco

Yet again we can overflow the input. Right at the end of `login`, immediately
before the `ret`, the stack is pushed ahead 16 bytes: `add #0x10, sp`.  `ret`,
[by definition](http://en.wikipedia.org/wiki/MSP430#Pseudo-operations), will set
the program counter to be whatever is on the stack.  Fiddling with different
large inputs you can see that it's easy to control what's on the stack
after the `add #0x10, sp`.  Again, it's bytes 17 and 18 of the input.  So we
just put the address of the `unlock_door` function (`0x4446`) there and voila.

My answer (hex): `aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa4644`

### Reykjavik

This one was interesting.  Only interesting function in the disassembly is
`enc`.  `main` calls `enc`, then calls whatever is at `0x2400`.  I grabbed the
memory from `0x2400` to `0x2570` (somewhat arbitrarily, I just grabbed a bunch
of memory) and used the [disassembler](https://microcorruption.com/assembler)
to see what it was doing.  You can also watch "current instruction" in the
debugger as you step, but I find the disassembler easier.  Anyway I wanted to
find the code that triggered the deadbolt, which involves interrupt `0x7f`.
i.e. look for `push 0x7f` in the disassembled output.  It occurs once, and its
guarded by a check that some location contains `0x8177`, i.e. `cmp #0x8177,
-0x24(r4)`.  Debugging through, location `-0x24(r4)` is the start of the input!
So this time it's a very secure two byte password.

My answer (hex): `7781`

### Johannesburg

### Whitehorse

