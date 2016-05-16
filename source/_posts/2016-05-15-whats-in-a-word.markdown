---
layout: post
title: "What's in a Word?"
date: 2016-05-15 17:58
comments: true
categories: x86 x86-64 word Assembly ISA
---
Recently, there some was some confusion between myself and a coworker over the
definition of a "word."  I'm currently working on a blog post about data
alignment and figured it would be good to clarify some things now, that we can
refer to later.

Having studied computer engineering and being quite fond of processor design,
when I think of a "word," I think of the number of bits wide a processor's
general purpose registers are
(aka [word size](https://en.wikipedia.org/wiki/Word_%28computer_architecture%29#Size_families)).
This places hard requirements on the largest representable number and address
space.  A 64 bit processor can represent 2^64-1 (1.8x10^19) as the largest
unsigned long integer, and address up to 2^64-1 (16 EiB) different addresses in
memory.

Further, word size limits the possible combinations of operations the processor
can perform, length of immediate values used, inflates the size of binary files
and memory needed to store pointers, and puts pressure on instruction caches.

Word size also has implications on loads and stores based on alignment, as
we'll see in a follow up post.

When I think of 8 bit computers, I think of my first microcontroller: an
Arduino with an Atmel AVR processor.  When I think of 16 bit computers, I think
of my first game console, a Super Nintendo with a Ricoh 5A22.  When I think of
32 bit computers, I think of my first desktop with Intel's Pentium III.  And
when I think of 64 bit computers, I think modern smartphones with ARMv8
instruction sets.  When someone mentions a particular word size, what are the
machines that come to mind for you?

So to me, when someone's talking about a 64b processor, to that machine (and
me) a word is 64b.  When we're referring to a 8b processor, a word is 8b.

Now, some confusion.

Back in my previous blog posts about
[x86-64 assembly](/blog/2014/04/18/lets-write-some-x86-64/),
[JITs](/blog/2015/05/25/interpreter-compiler-jit/), or
[debugging](/blog/2016/01/20/debugging-x86-64-assembly-with-lldb-and-dtrace/),
you might have seen me use instructions that have suffixes of b for byte (8b),
w for word (16b), dw for double word (32b), and qw for quad word (64b) (since
SSE2 there's also double quadwords of 128b).

Wait a minute!  How suddenly does a "word" refer to 16b on a 64b processor, as
opposed to a 64b "word?"

In short, historical baggage.  Intel's first hit processor was the
[4004](https://en.wikipedia.org/wiki/Intel_4004),
a 4b processor released in 1971.  It wasn't until 1979 that Intel created the
16b
[8086 processor](https://en.wikipedia.org/wiki/Intel_8086).

The 8086 was created to compete with other 16b processors that beat it to the
market, like the
[Zilog Z80](https://en.wikipedia.org/wiki/Zilog_Z80)
(any Gameboy emulator fans out there?  Yes, I know about the Sharp LR35902).
The 8086 was the first design in the
[x86 family](https://en.wikipedia.org/wiki/X86),
and it allowed for the same assembly syntax from the earlier 8008, 8080, and
8085 to be reassembled for it.  The 8086's little brother (8088) would be used
in
[IBM's PC](https://en.wikipedia.org/wiki/IBM_Personal_Computer#Open_standards),
and the rest is history.  x86 would become one of the most successful
ISAs in history.

For backwards compatibility, it seems that both Microsoft's (whose success has
tracked that of x86 since MS-DOS and IBM's PC) and Intel's documentation refers
to words still as being 16b. This allowed 16b PE32+ executables to be run on
32b or even 64b newer versions of Windows, without requiring recompilation of
source or source code modification.

This isn't necessarily wrong to refer to a word based on backwards
compatibility, it's just important to understand the context in which the term
"word" is being used, and that there might be some confusion if you have a
background with x86 assembly, Windows API programming, or processor design.

So the next time someone asks: why does Intel's documentation commonly refer to
a "word" as 16b, you can tell them that the x86 and x86-64 ISAs have maintained
the notion of a word being 16b since the first x86 processor, the 8086, which
was a 16b processor.

*Side Note: for an excellent historical perspective programming early x86
chips, I recommend Michael Abrash's*
[Graphics Programming Black Book](http://www.gamedev.net/page/resources/_/technical/graphics-programming-and-theory/graphics-programming-black-book-r1698).
*For instance he talks about 8086's little brother, the 8088, being a 16b chip
but only having an 8b bus with which to access memory. This caused a mysterious*
["cycle eater"](http://downloads.gamedev.net/pdf/gpbb/gpbb4.pdf)
*to prevent fast access to 16b variables, though they were the processor's
natural size.  Michael also alludes to alignment issues we'll see in a follow
up post.*

