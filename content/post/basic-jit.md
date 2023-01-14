+++
title = "Basic JIT"
date = "2013-04-03"
slug = "2013/04/03/basic-jit"
Categories = ["C", "Code", "JIT"]
+++
Ever since I learned about
[Just In Time Compilation](`http://en.wikipedia.org/wiki/Just-in-time_compilation)
from the various
[Ruby VMs](http://en.wikipedia.org/wiki/Ruby_%28programming_language%29#Implementations)
and
[JavaScript VMs](http://www.slideshare.net/newmovie/know-yourengines-velocity2011),
I've been inspired.  I could tell you all about how just in time (JIT)
compilation worked, and how it could give your interpreted language a speed
boost.  It was so cool.  Well, it still is!  There's a ton of research going on
around JIT compilation.  But the problem for me is that I could never figure
out, let alone guess, how it was being done.  How could you compile code at
runtime, and then execute it?  I asked
[Benjamin Peterson](http://pybites.blogspot.com/)
about how he had learned so much about JITs, and he referred me to
[pypy](http://pypy.org/) (a Python JIT)'s source.  But digging through source
was too much; I just wanted a simple example I could grok quickly.  So I've
always been left wondering.

Luckily, I have an awesome job where I can meet face to face with the people
who are doing the work on making awesome production JIT compilers.  I spent
part of last week at [GDC](http://www.gdconf.com/) demoing
[Unreal Engine 3](https://blog.mozilla.org/blog/2013/03/27/mozilla-is-unlocking-the-power-of-the-web-as-a-platform-for-gaming/)
running in the browser.  The demo was actually the hard work of many, and I'll
dive into it more in another post following up the events, but
[Luke Wagner](https://blog.mozilla.org/luke/), a Mozillian working on the
JavaScript engine, added OdinMonkey to SpiderMonkey to allow optimizations of
[asm.js](https://blog.mozilla.org/luke/2013/03/21/asm-js-in-firefox-nightly/).

Luke is super friendly, and just listening to him talk with Dave Herman and
Alon Zakai is a treat.  I asked Luke the basics of JIT'ing at tonight's Mozilla
Research Party and he explained clearly. *Compile a simple object file, use
objdump to get the resulting platform specific assembly, use the mmap system
call to allocate some memory that you can write to **AND** execute, copy the
instructions into that buffer, typecast it to a function pointer and finally 
call that.*

So my goal was to at runtime, create a function that for simplicity's sake
multiplied two integers.  The first thing I did was write up a simple .c file,
then compile that to an object file with the -c flag.

```c
// Compile me with: clang -c mul.c -o mul.o
int mul (int a, int b) {
  return a * b;
}
```

As a heads up, I'm on 64bit OSX.  So the generated assembly may differ on your
platform.  Obviously, the production JIT maintainers have abstracted away the
platform dependency, taking into account what platform you're on.  My example
doesn't, but that why I'm walking you through the steps I took to find out the
x86\_64 instructions.  The next step is to grab binutils, which is not installed
by default in OSX.  I used homebrew to install it: `brew install binutils`.
Homebrew installs gobjdump but it works with the same flags.

Once you have binutils and [g]objdump, the next step is to read out the machine
code from your object file, which is represented in hexadecimal.  By running
`gobjdump -j .text -d mul.o -M intel` you should get something similar
(remember, architecture dependent).

```
$ gobjdump -j .text -d mul.o -M intel

Disassembly of section .text:

0000000000000000 <_mul>:
0: 55 push rbp
1: 48 89 e5 mov rbp,rsp
4: 89 7d fc mov DWORD PTR [rbp-0x4],edi
7: 89 75 f8 mov DWORD PTR [rbp-0x8],esi
a: 8b 75 fc mov esi,DWORD PTR [rbp-0x4]
d: 0f af 75 f8 imul esi,DWORD PTR [rbp-0x8]
11: 89 f0 mov eax,esi
13: 5d pop rbp
14: c3 ret
```

Ok, so those instructions vary in size.  I don't know any x86 so I can't
comment too much on that particular
[Instruction Set Architecture](http://en.wikipedia.org/wiki/Instruction_set)
but they're obviously in pairs of hexadecimal digits.  16^2 == 2^8 meaning that
each pair of hex digits can be represented by a single byte (or char of memory).
So these can all be thrown in an unsigned char [].  The man page for mmap explains
all of the fun flags, but the important point is that this way of allocating
memory makes it executable, so it can do bad things that memory allocated from
malloc can't.  I'm sure the JavaScript engine guys have fun with that.  Once
memory is copied in, you can typecast the memory to a function pointer.
Make sure to check out the syntax that reminds me of a function pointer in an
argument list, but being used as an L-value.  Of course, you could just put the
cast right in front of the memory before you use it, but I find this as neat,
not so common C syntax.  I kind of
[have a thing](http://nickdesaulniers.github.com/blog/2013/01/26/c-function-pointers-alternate-syntax/)
for stuff like that.  Then we can call it!  The resulting code looks like this.

```c
#include <stdio.h> // printf
#include <string.h> // memcpy
#include <sys/mman.h> // mmap, munmap

int main () {
// Hexadecimal x86_64 machine code for: int mul (int a, int b) { return a * b; }
unsigned char code [] = {
  0x55, // push rbp
  0x48, 0x89, 0xe5, // mov rbp, rsp
  0x89, 0x7d, 0xfc, // mov DWORD PTR [rbp-0x4],edi
  0x89, 0x75, 0xf8, // mov DWORD PTR [rbp-0x8],esi
  0x8b, 0x75, 0xfc, // mov esi,DWORD PTR [rbp-04x]
  0x0f, 0xaf, 0x75, 0xf8, // imul esi,DWORD PTR [rbp-0x8]
  0x89, 0xf0, // mov eax,esi
  0x5d, // pop rbp
  0xc3 // ret
};

  // allocate executable memory via sys call
  void* mem = mmap(NULL, sizeof(code), PROT_WRITE | PROT_EXEC,
                   MAP_ANON | MAP_PRIVATE, -1, 0);

  // copy runtime code into allocated memory
  memcpy(mem, code, sizeof(code));

  // typecast allocated memory to a function pointer
  int (*func) () = mem;

  // call function pointer
  printf("%d * %d = %d\n", 5, 11, func(5, 11));

  // Free up allocated memory
  munmap(mem, sizeof(code));
}
```

Voila!  How neat is that!  It works!  Luke's directions, an hour of working on
this, and
[this article](http://blog.reverberate.org/2012/12/hello-jit-world-joy-of-simple-jits.html)
in particular, and we have a simple JIT working. Again, this simple example is super
non-portable and dealing with memory in this fashion is generally unsafe.  But
now I know the basics of JIT'ing code, and now so do you!

Thanks Luke!
