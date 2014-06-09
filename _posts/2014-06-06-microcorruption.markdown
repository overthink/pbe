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
challenge yourself!  It'll only take... tens of hours!  But it's a lot of fun.

## Answers

Disclaimer: I don't know what I'm doing, as usual.  I passed each level, but my
reasoning exposed in the writeups might be flawed. Feel free to enlighten me if
you see problems.

### Tutorial (10)

Well, they lead you through this one.  In the `check_password` function you can
see it incrementing a count in `r12` after each character of the password is
read.  Then if the counter is equal to nine (meaning eight characters read)
`cmp #0x9, r12`, it signals the password is good (`mov #0x1, r15`) and we're
done.

My answer: `aaaaaaaa` (any eight-char password)

### New Orleans (10)

The first thing done in `main` is a call to `create_password` which promptly
puts the password into main memory at address `0x2400`.  Then you just look at
it.  Doesn't get a lot easier.

My answer: `z99zlHo`

### Sydney (15)

This one just compares the bytes in the entered password to a bunch of
constants in code.  You can read the password right out of `check_password`.
If you're like me and have no idea what you're doing, you'll likely trip up on
endianness here.  The first check in `check_password` is `cmp #0x3024,
0x0(r15)`, meaning the first two bytes are `30` and `24` (hex).  But the MSP430
is little-endian, so the least significant byte (`24`) is stored in the lower
(smaller) memory address.  So you have to enter `2430` (hex) for the first two
bytes.  Same procedure for the next six bytes.

My answer (hex): `2430426b5466706d`

### Hanoi (20)

What happens right before the call to `unlock_door` in the `login` procedure?
Checking that the contents of a fixed address are the constant `0x3` (i.e.
`cmp.b #0x3, &0x2410`).  Working backwards I need to get `0x3` into address
`0x2410`.  Fortunately, you can overflow the input here.  Passwords are stored
starting at `0x2400` and while they're supposedly not to exceed 16 chars in
length, you can enter up to 28.  Why?  They pass `0x1c` (decimal 28) as the
second arg to interrupt `0x02` in `getsn`, meaning it'll read 28 bytes of
input.  Anyway, all we need to do is make sure the 17th byte of the password,
which will be stored at `0x2410` is 0x3.

My answer (hex): `aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa0300`

### Cusco (25)

Yet again we can overflow the input. Right at the end of `login`, immediately
before the `ret`, the stack is pushed ahead 16 bytes: `add #0x10, sp`.  `ret`,
[by definition](http://en.wikipedia.org/wiki/MSP430#Pseudo-operations), will set
the program counter to be whatever is on the stack.  Fiddling with different
large inputs you can see that it's easy to control what's on the stack
after the `add #0x10, sp`.  Again, it's bytes 17 and 18 of the input.  So we
just put the address of the `unlock_door` function (`0x4446`) there and voila.

My answer (hex): `aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa4644`

### Reykjavik (35)

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

### Johannesburg (20)

This one took me hours and hours, despite it being almost trivial.  Lesson: look at
the WHOLE program, don't assume the exploit will be in a particular place just
because it was in previous levels.  With blinders off you'll notice code at the
end of `login` that checks the length of the entered password and outputs a
"too long" message if the password was too long.  It does this by checking for
a sentinel value (`0xc8`) in memory that ends up overwritten if the password is too
long `cmp.b #0xc8, 0x11(sp)`.  This might be ok except that:

* the program always reads 63 (`0x3f`) bytes of user input (`mov #0x3f, r14` in
  `login`)
* the program then copies what was read using `strcpy` to a "sensitive" part
  of memory near the stack pointer assuming it won't exceed 16 bytes
* `strcpy` will copy any size of bytes, assuming they're a null-terminated
  string

This means we can craft a password to manipulate what is on the stack after the
"too long" message is printed. If the password length check succeeds it moves
the stack pointer to the word right after the sentinel value (i.e. the 19th and
20th bytes of input) and executes `ret`.  So you enter a 20 byte password where
byte 17 is the sentinel `0xc8` and bytes 19 and 20 are the address of the
`unlock_door` function.

My answer (hex): `aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaac84644`

### Whitehorse (50)

Once again, the program reads too much input.  This time they read 48 bytes
(`mov #0x30, r14` in `login`).  This is notable since after a failed password
the stack is advanced 16 bytes from where the password starts, and `ret` is
executed.  So once again we take control of the program counter by writing
whatever return address we want onto the stack.  The twist here is that there's
no unlock function to jump to.  What we can do, however, is "return" to the
address of where our password is stored.  Then we just encode instructions to
open the lock in our password.  I supposed this is "arbitrary code execution".
Reading the manual we can see that interrupt `0x7f` will open the lock, so we
encode the following into our password, using the handy
[assembler](https://microcorruption.com/assembler):

  push #0x7f
  call #0x4532 ; The address of the INT function

This assembles to `30127f00b0123245`, and the password is stored at `0x3c56`.
Combine these for the answer:

My answer (hex): `30127f00b01232450000000000000000563c`

