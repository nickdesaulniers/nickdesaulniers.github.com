+++
title = "Data Models and Word Size"
date = "2016-05-30"
slug = "2016/05/30/data-models-and-word-size"
Categories = []
+++
*This post is a follow up to
[my previous blog post about word size](/blog/2016/05/15/whats-in-a-word/).*

Three C/C++ programmers walk into a bar.  One argues that sizeof(void\*) is
equivalent to sizeof(long), one argues that sizeof(void\*) is equivalent to
sizeof(int), and the third argues it’s sizeof(long long).  Simultaneously,
they’re all right, but they’re also all wrong (and need a lesson about portable
C code).  What the hell is going on?

One of the first few programs a programmer might write after hello world is
something like this:

```c
#include <stdio.h>
int main () {
  printf("sizeof(int): %zu\n", sizeof(int));
  printf("sizeof(long): %zu\n", sizeof(long));
  printf("sizeof(long long): %zu\n", sizeof(long long));
  printf("sizeof(void*): %zu\n", sizeof(void*));
}
```

*Note the use of the %zu format specifier, a C99 addition that isn’t portable to
older compilers!  (This post is more about considerations when porting older
code to newer machines, not about porting newer code to run on older machines.
Not having a standards compliant C compiler makes writing more portable C code
even trickier, and is a subject for another blog post).*

When I run that code on my x86-64 OSX machine, I get the following output:

```sh
sizeof(int): 4
sizeof(long): 8
sizeof(long long): 8
sizeof(void*): 8
```

So it looks like I would be the first programmer in the story in the first
paragraph, since on my machine, it looks like sizeof(long) == sizeof(void\*).
Also note how sizeof(long long) is equivalent as well.

But what would happen if I compiled my code on a 32 bit machine?  Luckily, my
processor has backwards compatibility with 32b binaries, so I can cross compile
it locally and still run it. Ex:

```sh
➜  clang sizeof.c -Wall -Wextra -Wpedantic
➜  file a.out
a.out: Mach-O 64-bit executable x86_64
➜  clang sizeof.c -Wall -Wextra -Wpedantic -m32
➜  file a.out
a.out: Mach-O executable i386
➜  ./a.out
sizeof(int): 4
sizeof(long): 4
sizeof(long long): 8
sizeof(void*): 4
```

Huh, suddenly sizeof(void\*) == sizeof(int) == sizeof(long)!  This seems
to be the case of the second programmer from the story.

Both programmer 1 and programmer 2 might agree that the size of a pointer is
equivalent to their machine’s respective
[word size](/blog/2016/05/15/whats-in-a-word/),
but that too would be an incorrect assumption for portable C/C++ code!

Programmer 3 goes through the hellscape that is installing a working compiler
for Windows and building a 64b command line application (to be fair, installing
command line tools for OSX is worse; installing a compiler for most OS’ leaves
much to be desired).  When they run that program, they see:

```
sizeof(int): 4
sizeof(long): 4
sizeof(long long): 8
sizeof(void*): 8
```

This is yet a third case (the third programmer from the story).  In this case,
only sizeof(long long) is equivalent to sizeof(void\*).

###Data Models

What these programmers are seeing is known as data models.  Programmer 1 one on
a 64b x86-64 OSX machine had an LP64 data model where longs (L), (larger long
longs,) and pointers (P) are 64b, but ints were 32b.  Programmer 2 on a 32b x86
OSX machine had an ILP32 data model where ints (I), longs (L), and pointers (P)
were 32b, but long longs were 64b.  Programmer 3 on a 64b x86-64 Windows
machine had a LLP64 data model, where only long longs (LL) and pointers (P)
were 64b, ints and longs being 32b.

**Data model** | **sizeof(int)** | **sizeof(long)** | **sizeof(long long)** | **sizeof(void\*)** | **example**
--- | --- | --- | --- | --- | ---
ILP32 | 32b | 32b | 64b | 32b | Win32, i386 OSX & Linux
LP64 | 32b | 64b | 64b | 64b | x86-64 OSX & Linux
LLP64 | 32b | 32b | 64b | 64b | Win64

There are older data models such as LP32 (Windows 3.1, Macintosh, where ints
are 16b), and more exotic ones like ILP64, and SILP64.  Knowing the data model
thus is important for portable C/C++ code.

###Historical Perspective

Running out of address space is and will continue to be tradition in computing.
Applications become bigger as computer power and memory gets cheaper.
Companies want to sell chips that have larger word sizes to address more
memory, but early adopters don’t want to buy a computer where there favorite
application hasn’t been compiled and thus doesn’t exist on yet.  **Someone from
the back shouts *virtual machines* then ducks as a chair is thrown.**

[This document](http://www.unix.org/version2/whatsnew/lp64_wp.html)
highlights some reasons why LP64 is preferred to ILP64: ILP64
made portable C code that only needed 32b of precision harder to maintain (on
ILP64 an int was 64b, but a short was 16b!).  It mentions how for data
structures that did not contain pointers, their size would be the same on LLP64
as ILP32, which is the direction Microsoft went.  LLP64 was essentially the
ILP32 model with 64b pointers.

*Linux also supports an ABI called
[x32](https://en.wikipedia.org/wiki/X32_ABI)
which can use x86-64 ISA improvements but uses 32b pointers to reduce the size
of data structures that would otherwise have 64b pointers.*

For a great historical perspective on the evolution of word size and data
models, as well as the "toil and trouble" caused,
[this paper](https://queue.acm.org/detail.cfm?id=1165766)
was an excellent reference.  It describes Microsoft finally abandoning support
for 16b data models in Windows XP 64.  It mentions that the industry was pretty
split between LP64, LLP64, and ILP64 as porting code from the good old days of
ILP32 would break in different ways.  That the use of long was more prevalent
in Windows applications vs the use of int in unix applications.  It also makes
the key point that a lot of programmers from the ILP32 era made assumptions
that sizeof(int) == sizeof(long) == sizeof(void\*) which would not hold true
for the LP64/LLP64 era.

One important point the paper makes makes that’s easily missed is that typedef
wasn’t added to C until 1977 when hardware manufactures still couldn’t agree on
how many bits were in a char (CHAR\_BITS) and some machines were using 24b
addressing schemes.  stdint.h and inttypes.h did not exist yet.

[This article](/blog/2016/05/15/whats-in-a-word/)
talks about two main categories of effects of switching from ILP32 to LP64 and
has excellent examples of problematic code.  That section near the end is worth
the read alone and makes excellent points to look for during code review.

###Conclusion

Word size or ISA doesn’t tell you anything about sizeof(int), sizeof(long), or
sizeof(long long).  We also saw that one machine can support multiple different
data models (when I compiled and ran the same code with the -m32 flag).

The C standard tells you minimum guaranteed sizes for these types, but the data
model (part of the ABI, external to but abiding by the C standard) is what
tells you about the specifics sizes of standard integers and pointers.

###Further Reading
* [64-Bit Programming Models: Why LP64?](http://www.unix.org/version2/whatsnew/lp64_wp.html)
* [The Long Road to 64 Bits](https://queue.acm.org/detail.cfm?id=1165766)
* [The UNIX System -- 64bit and Data Size Neutrality](http://www.unix.org/whitepapers/64bit.html)
* [64-bit data models](https://en.wikipedia.org/wiki/64-bit_computing#64-bit_data_models)
* [C Language Data Type Models: LP64 and ILP32](https://docs.oracle.com/cd/E19620-01/805-3024/lp64-1/index.html)
* [ILP64, LP64, LLP64](https://blogs.oracle.com/nike/entry/ilp64_lp64_llp64)
* [x32 ABI](https://en.wikipedia.org/wiki/X32_ABI)
* [difference between stdint.h and inttypes.h](http://stackoverflow.com/a/9162072)
* [Abstract Data Models](https://msdn.microsoft.com/en-us/library/windows/desktop/aa384083%28v=vs.85%29.aspx)
* [The New Data Types](https://msdn.microsoft.com/en-us/library/windows/desktop/aa384264%28v=vs.85%29.aspx)
* [Is there any reason not to use fixed width integer types (e.g. uint8_t)?](http://stackoverflow.com/a/13413892)
* [Why did the Win64 team choose the LLP64 model?](https://blogs.msdn.microsoft.com/oldnewthing/20050131-00/?p=36563/)

