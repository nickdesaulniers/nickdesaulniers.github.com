---
layout: post
title: "Object Files and Symbols"
date: 2016-08-13 20:46
comments: true
categories: Object file Symbol C C++ Demangle otool nm objdump
---
What was supposed to be one blog post about memory segmentation turned into
what will be a series of posts.  As the first in the series, we cover the
extreme basics of object files and symbols.  In follow up posts, I
plan to talk about static libraries, dynamic libraries, dynamic linkage, memory
segments, and finally memory usage accounting.  I also cover command line tools
for working with these notions, both in Linux and OSX.

A quick review of the compilation+execution pipeline (for terminology):

1. Lexing produces tokens
2. Parsing produces an abstract syntax tree
3. Analysis produces a code flow graph
4. Optimization produces a reduced code flow graph
5. Code gen produces object code
6. Linkage produces a complete executable
7. Loader instructs the OS how to start running the executable

This series will focus on part #6.

Let's say you have some amazing C/C++ code,  but for separations of concerns,
you want to start moving it out into separate source files.  Whereas previously
in one file you had:

```c
// main.c
#include <stdio.h>
void helper () {
  puts("helper");
}
int main () {
  helper();
}
```

You now have two source files and maybe a header:

```c
// main.c
#include "helper.h"
int main () {
  helper();
}

// helper.h
void helper();

//helper.c
#include <stdio.h>
#include "helper.h"
void helper () {
  puts("helper");
}
```

In the single source version, we would have compiled and linked that with
`clang main.c` and had an executable file.  In the multiple source version, we
first compile our source files to object files, then link them altogether.
That can be done separately:

```sh
$ clang -c helper.c     # produces helper.o
$ clang -c main.c       # produces main.o
$ clang main.o helper.o # produces a.out
```

We can also do the compilation and linkage in one step:

```sh
$ clang helper.c main.c # produces a.out
```

Nothing special thus far; C/C++ 101.  In the first case of separate compilation
and linkage steps, we were left with intermediate object files (.o).  What
exactly are these?

[Object files](https://en.wikipedia.org/wiki/Object_file)
are almost full executables.  They contain machine code, but that code still
requires a relocation step.  It also contains metadata about the addresses of
its variables and functions (called symbols) in an associative data structure
called a
[symbol table](https://en.wikipedia.org/wiki/Symbol_table).
The addresses may not be the final address of the symbol in the final
executable. They also contain some information for the loader and probably some
other stuff.

Remember that if we fail to specify the helper object file, we'll get an
undefined symbol error.

```sh
$ clang main.c
Undefined symbols for architecture x86_64:
  "_helper", referenced from:
      _main in main-459dde.o
ld: symbol(s) not found for architecture x86_64
clang: error: linker command failed with exit code 1 (use -v to see invocation)
```

The problem is main.o refers to some symbol called `helper`, but on it's own
doesn't contain any more information about it.  Let's say we want to know what
symbols an object file contains, or expects to find elsewhere.  Let's introduce
our first tool, `nm`.  `nm` will print the name list or symbol table for a
given object or executable file.  On OSX, these are prefixed with an
underscore.

```sh
$ nm helper.o
0000000000000000 T _helper
                 U _puts

$ nm main.o
                 U _helper
0000000000000000 T _main

$ nm a.out
...
0000000100000f50 T _helper
0000000100000f70 T _main
                 U _puts
...
```

Let's dissect what's going on here.  The output (as understood by `man 1 nm`)
is a space separated list of address, type, and symbol name.  We can see that
the addresses are placeholders in object files, and final in executables.  The
name should make sense; it's the name of the function or variable.  While I'd
love to get in depth on the various symbol types and talk about sections, I
don't think I could do as great a job as Peter Van Der Linden in his book
"Expert C Programming: Deep C Secrets."

For our case, we just care about whether the symbol in a given object file is
defined or not.  The type U (undefined) means that this symbol is referenced or
used in this object code/executable, but it's value wasn't defined here.
When we compiled main.c alone and got the undefined symbol error, it should now
make sense why we got the undefined symbol error for helper.  main.o contains
a symbol for main, and references helper.  helper.o contains a symbol for
helper, and references to puts.  The final executable contains symbols for main
and helper and references to puts.

You might be wondering where puts comes from then, and why didn't we get an
undefined symbol error for puts like we did earlier for helper.  The answer is
the C runtime.  libc is implicitly dynamically linked to all executables
created by the C compiler.  We'll cover dynamic linkage in a later post in
this series.

When the linker performs relocation on the object files, combining them into a
final executable, it goes through placeholders of addresses and fills them in.
We did this manually in our post on
[JIT compilers](/blog/2015/05/25/interpreter-compiler-jit/).

While `nm` gave us a look into our symbol table, two other tools I use
frequently are `objdump` on Linux and `otool` on OSX.  Both of these provide
disassembled assembly instructions and their addresses.  Note how the symbols
for functions get translated into labels of the disassembled functions, and
that their address points to the first instruction in that label.  Since I've
shown `objdump`
[numerous times](/blog/2013/04/03/basic-jit/)
in
[previous posts](/blog/2014/04/18/lets-write-some-x86-64/),
here's `otool`.

```sh
$ otool -tV helper.o
helper.o:
(__TEXT,__text) section
_helper:
0000000000000000    pushq    %rbp
0000000000000001    movq    %rsp, %rbp
0000000000000004    subq    $0x10, %rsp
0000000000000008    leaq    0xe(%rip), %rdi         ## literal pool for: "helper"
000000000000000f    callq    _puts
0000000000000014    movl    %eax, -0x4(%rbp)
0000000000000017    addq    $0x10, %rsp
000000000000001b    popq    %rbp
000000000000001c    retq
$ otool -tV main.o
main.o:
(__TEXT,__text) section
_main:
0000000000000000    pushq    %rbp
0000000000000001    movq    %rsp, %rbp
0000000000000004    movb    $0x0, %al
0000000000000006    callq    _helper
000000000000000b    xorl    %eax, %eax
000000000000000d    popq    %rbp
000000000000000e    retq
$ otool -tV a.out
a.out:
(__TEXT,__text) section
_helper:
0000000100000f50    pushq    %rbp
0000000100000f51    movq    %rsp, %rbp
0000000100000f54    subq    $0x10, %rsp
0000000100000f58    leaq    0x43(%rip), %rdi        ## literal pool for: "helper"
0000000100000f5f    callq    0x100000f80             ## symbol stub for: _puts
0000000100000f64    movl    %eax, -0x4(%rbp)
0000000100000f67    addq    $0x10, %rsp
0000000100000f6b    popq    %rbp
0000000100000f6c    retq
0000000100000f6d    nop
0000000100000f6e    nop
0000000100000f6f    nop
_main:
0000000100000f70    pushq    %rbp
0000000100000f71    movq    %rsp, %rbp
0000000100000f74    movb    $0x0, %al
0000000100000f76    callq    _helper
0000000100000f7b    xorl    %eax, %eax
0000000100000f7d    popq    %rbp
0000000100000f7e    retq
```

`readelf -s <object file>` will give us a list of symbols on Linux.
[ELF](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format)
is the file format used by the loader on Linux, while OSX uses
[Mach-O](https://en.wikipedia.org/wiki/Mach-O).
Thus `readelf` and `otool`, respectively.

Also note that for static linkage, symbols need to be unique*, as they refer to
memory locations to either read/write to in the case of variables or locations
to jump to in the case of functions.

```sh
$ cat double_define.c
void a () {}
void a () {}
int main () {}
$ clang double_define.c
double_define.c:2:6: error: redefinition of 'a'
void a () {}
     ^
double_define.c:1:6: note: previous definition is here
void a () {}
     ^
1 error generated.
```

*: there's a notion of weak symbols, and some special things for dynamic
libraries we'll see in a follow up post.

Languages like C++ that support function overloading (functions with the same
name but different arguments, return types, namespaces, or class) must mangle
their function names to make them unique.

Code like:
```c++
namespace util {
  class Widget {
    public:
      void doSomething (bool save);
      void doSomething (int n);
  };
}
```
Will produce symbols like:
```sh
$ clang class.cpp -std=c++11
$ nm a.out
0000000100000f70 T __ZN4util6Widget11doSomethingEb
0000000100000f60 T __ZN4util6Widget11doSomethingEi
...
```
Note: GNU `nm` on Linux distros will have a `--demangle` option:
```sh
$ nm --demangle a.out
...
00000000004006d0 T util::Widget::doSomething(bool)
00000000004006a0 T util::Widget::doSomething(int)
...
```
On OSX, we can pipe `nm` into `c++filt`:
```sh
$ nm a.out | c++filt
0000000100000f70 T util::Widget::doSomething(bool)
0000000100000f60 T util::Widget::doSomething(int)
...
```
Finally, if you don't have an object file, but instead a backtrace that needs
demangling, you can either invoke `c++filt` manually or use
[demangler.com](http://demangler.com/).

Rust also mangles its function names.  For FFI or interface with C functions,
other languages usually have to look for or expose symbols in a manner suited
to C, the lowest common denominator.
[C++](http://en.cppreference.com/w/cpp/language/language_linkage)
has `extern "C"` blocks and
[Rust](https://doc.rust-lang.org/book/ffi.html)
has `extern` blocks.

We can use `strip` to remove symbols from a binary.  This can slim down a
binary at the cost of making stack traces unreadable.  If you're following
along at home, try comparing the output from your disassembler and `nm` before
and after running `strip` on the executable.  Luckily, you can't strip the
symbols out of object files, otherwise they'd be useless as you'd no longer be
able to link them.

If we compile with the `-g` flag, we can create a different kind of symbol;
[debug symbols](https://en.wikipedia.org/wiki/Debug_symbol).
Depending on your compiler+host OS, you'll get another file you can run through
`nm` to see an entry per symbol.  You'll get more info by using `dwarfdump` on
this file.  Debug symbols will retain source information such as filename and
line number for all symbols.

This post should have been a simple refresher of some of the basics of working
with C code. Finding symbols to be placed into a final executable and
relocating addresses are the main job of the linker, and will be the main theme
of the posts in this series. Keep your eyes out for more in this series on
memory segmentation.

