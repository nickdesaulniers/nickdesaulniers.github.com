+++
title = "Static and Dynamic Libraries"
date = "2016-11-20"
slug = "2016/11/20/static-and-dynamic-libraries"
Categories = []
+++
This is the second post in a series on memory segmentation.  It covers working
with static and dynamic libraries in Linux and OSX.  Make sure to check out the
[first on object files and symbols](/blog/2016/08/13/object-files-and-symbols/).


Let’s say we wanted to reuse some of the code from our previous project in our
next one.  We could continue to copy around object files, but let’s say we have
a bunch and it’s hard to keep track of all of them.  Let’s combine multiple
object files into an archive or static library.  Similar to a more conventional
zip file or "compressed archive," our static library will be an uncompressed
archive.


We can use the `ar` command to create and manipulate a static archive.


```sh
$ clang -c x.c y.c
$ ar -rv libhello.a x.o y.o
```


The `-r` flag will create the archive named `libhello.a` and add the files
`x.o` and `y.o` to its index.  I like to add the `-v` flag for verbose output.
Then we can use the familiar `nm` tool I introduced in the
[previous post](/blog/2016/08/13/object-files-and-symbols/)
to examine the content of the archives and their symbols.


```sh
$ file libhello.a
libhello.a: current ar archive random library
$ nm libhello.a
libhello.a(x.o):
                 U _puts
0000000000000000 T _x


libhello.a(y.o):
                 U _puts
0000000000000000 T _y
```


Some other useful flags for `ar` are `-d` to delete an object file, ex. `ar -d
libhello.a y.o` and `-u` to update existing members of the archive when their
source and object files are updated.  Not only can we run `nm` on our archive,
`otool` and `objdump` both work.


Now that we have our static library, we can statically link it to our program
and see the resulting symbols.  The `.a` suffix is typical on both OSX and
Linux for archive files.


```sh
$ clang main.o libhello.a
$ nm a.out
0000000100000f30 T _main
                 U _puts
0000000100000f50 T _x
0000000100000f70 T _y
```

Our compiler understands how to index into archive files and pull out the
functions it needs to combine into the final executable.  If we use a static
library to statically link all functions required, we can have one binary with
no dependencies.  This can make deployment of binaries simple, but also greatly
increase their size.  Upgrading large binaries incrementally becomes more
costly in terms of space.


While static libraries allowed us to reuse source code, static linkage does not
allow us to reuse memory for executable code between different processes.  I
really want to put off talking about memory benefits until the next post, but
know that the solution to this problem lies in "dynamic libraries."


While having a single binary file keeps things simple, it can really hamper
memory sharing and incremental relinking.  For example, if you have multiple
executables that are all built with the same static library, unless your OS is
really smart about copy-on-write page sharing, then you’re likely loading
multiple copies of the same exact code into memory! What a waste!  Also, when
you want to rebuild or update your binary, you spend time performing relocation
again and again with static libraries.  What if we could set aside object files
that we could share amongst multiple instances of the same or even different
processes, and perform relocation at runtime?


The solution is known as dynamic libraries.  If static libraries and static
linkage were Atari controllers, dynamic libraries and dynamic linkage are Steel
Battalion controllers.  We’ll show how to work with them in the rest of this
post, but I’ll prove how memory is saved in a later post.


Let’s say we want to created a shared version of libhello.  Dynamic libraries
typically have different suffixes per OS since each OS has it’s preferred
object file format.  On Linux the .so suffix is common, .dylib on OSX, and .dll
on Windows.


```sh
$ clang -shared -fpic x.c y.c -o libhello.dylib
$ file libhello.dylib
libhello.dylib: Mach-O 64-bit dynamically linked shared library x86_64
$ nm libhello.dylib
                 U _puts
0000000000000f50 T _x
0000000000000f70 T _y
```

The `-shared` flag tells the linker to create a special file called a shared
library.  The `-fpic` option converts absolute addresses to relative addresses,
which allows for different processes to load the library at different virtual
addresses and share memory.

Now that we have our shared library, let’s dynamically link it into our
executable.

```sh
$ clang main.c libhello.dylib
$ ./a.out
x
y
```

The dynamic linker essential produces an incomplete binary.  You can verify
with `nm`.  At runtime, we’ll delay start up to perform some memory mapping
early on in the process start (performed by the dynamic linker) and pay slight
costs for trampolining into position independent code.

Let’s say we want to know what dynamic libraries a binary is using.  You can
either query the executable (most executable object file formats contain a
header the dynamic linker will parse and pull in libs) or observe the
executable while running it.  Because each major OS has its own object file
format, they each have their own tools for these two checks.  Note that
statically linked libraries won’t show up here, since their object code has
already been linked in and thus we’re not able to differentiate between object
code that came from our first party code vs third party static libraries.

On OSX, we can use `otool -L <bin>` to check which .dylibs will get pulled in.

```sh
$ otool -L a.out
a.out:
           libhello.dylib (compatibility version 0.0.0, current version 0.0.0)
           /usr/lib/libSystem.B.dylib (compatibility version 1.0.0, current version 1226.10.1)
```

So we can see that `a.out` depends on `libhello.dylib` (and expects to find it
in the same directory as `a.out`).  It also depends on shared library called
libSystem.B.dylib.  If you run `otool -L` on libSystem itself, you’ll see it
depends on a bunch of other libraries including a C runtime, malloc
implementation, pthreads implementation, and more.  Let’s say you want to find
the final resting place of where a symbol is defined, without digging with `nm`
and `otool`, you can fire up your trusty debugger and ask it.

```sh
$ lldb a.out
...
(lldb) image lookup -r -s puts
...
        Summary: libsystem_c.dylib`puts        Address: libsystem_c.dylib[0x0000000000085c30] (libsystem_c.dylib.__TEXT.__stubs + 3216)
```

You’ll see a lot of output since `puts` is treated as a regex.  You’re looking
for the Summary line that has an address and is **not** a symbol stub.  You can
then check your work with `otool` and `nm`.


If we want to observe the dynamic linker in action on OSX, we can use `dtruss`:

```sh
$ sudo dtruss ./a.out
...
stat64("libhello.dylib\0", 0x7FFF50CEAC68, 0x1)         = 0 0
open("libhello.dylib\0", 0x0, 0x0)              = 3 0
...
mmap(0x10EF27000, 0x1000, 0x5, 0x12, 0x3, 0x0)          = 0x10EF27000 0
mmap(0x10EF28000, 0x1000, 0x3, 0x12, 0x3, 0x1000)               = 0x10EF28000 0
mmap(0x10EF29000, 0xC0, 0x1, 0x12, 0x3, 0x2000)         = 0x10EF29000 0
...
close(0x3)              = 0 0
...
```

On Linux, we can simply use `ldd` or `readelf -d` to query an executable for a
list of its dynamic libraries.


```sh
$ clang -shared -fpic x.c y.c -o libhello.so
$ clang main.c libhello.so
$ ldd a.out
           linux-vdso.so.1 =>  (0x00007fff95d43000)
           libhello.so => not found
           libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007fcc98c5f000)
           /lib64/ld-linux-x86-64.so.2 (0x0000555993852000)
$ readelf -d a.out
Dynamic section at offset 0xe18 contains 25 entries:
  Tag        Type                         Name/Value
 0x0000000000000001 (NEEDED)             Shared library: [libhello.so]
 0x0000000000000001 (NEEDED)             Shared library: [libc.so.6]
...
```

We can then use `strace` to observe the dynamic linker in action on Linux:

```sh
$ LD_LIBRARY_PATH=. strace ./a.out
...
open("./libhello.so", O_RDONLY|O_CLOEXEC) = 3
...
read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\260\5\0\0\0\0\0\0"..., 832) = 832
fstat(3, {st_mode=S_IFREG|0755, st_size=8216, ...}) = 0
close(3)                                = 0
...
```

What’s this `LD_LIBRARY_PATH` thing?  That’s shell syntax for setting an
environmental variable just for the duration of that command (as opposed to
exporting it so it stays set for multiple commands).  As opposed to OSX’s
dynamic linker, which was happy to look in the cwd for libhello.dylib, on Linux
we must supply the cwd if the dynamic library we want to link in is not in the
standard search path.

But what is the standard search path?  Well, there’s another environmental
variable we can set to see this, `LD_DEBUG`.  For example, on OSX:

```sh
$ LD_DEBUG=libs LD_LIBRARY_PATH=. ./a.out
     15828:        find library=libhello.so [0]; searching
     15828:         search path=./tls/x86_64:./tls:./x86_64:.             (LD_LIBRARY_PATH)
     15828:          trying file=./tls/x86_64/libhello.so
     15828:          trying file=./tls/libhello.so
     15828:          trying file=./x86_64/libhello.so
     15828:          trying file=./libhello.so
     15828:
     15828:        find library=libc.so.6 [0]; searching
     15828:         search path=./tls/x86_64:./tls:./x86_64:.             (LD_LIBRARY_PATH)
     15828:          trying file=./tls/x86_64/libc.so.6
     1earc:          trying file=./tls/libc.so.6
     15828:          trying file=./x86_64/libc.so.6
     15828:          trying file=./libc.so.6
     15828:         search cache=/etc/ld.so.cache
     15828:          trying file=/lib/x86_64-linux-gnu/libc.so.6
     15828:        calling init: /lib/x86_64-linux-gnu/libc.so.6
     15828:        calling init: ./libhello.so
     15828:        initialize program: ./a.out
     15828:        transferring control: ./a.out
x
y
     15828:        calling fini: ./a.out [0]
     15828:        calling fini: ./libhello.so [0]
```

`LD_DEBUG` is pretty useful.  Try:

```sh
$ LD_DEBUG=help ./a.out
Valid options for the LD_DEBUG environment variable are:


  libs        display library search paths
  reloc       display relocation processing
  files       display progress for input file
  symbols     display symbol table processing
  bindings    display information about symbol binding
  versions    display version dependencies
  scopes      display scope information
  all         all previous options combined
  statistics  display relocation statistics
  unused      determined unused DSOs
  help        display this help message and exit


To direct the debugging output into a file instead of standard output
a filename can be specified using the LD_DEBUG_OUTPUT environment variable.
```
For some cool stuff, I recommend checking out `LD_DEBUG=symbols` and
`LD_DEBUG=statistics`.

Going back to `LD_LIBRARY_PATH`, usually libraries you create and want to reuse
between projects go into /usr/local/lib and the headers into
/usr/local/include.  I think of the convention as:

```sh
$ tree -L 2 /usr/
/usr
├── bin # system installed binaries like nm, gcc
├── include # system installed headers like stdio.h
├── lib # system installed libraries, both static and dynamic
└── local
    ├── bin # user installed binaries like rustc
    ├── include # user installed headers
    └── lib # user installed
```

Unfortunately, it’s a loose convention that’s broken down over the years and
things are scattered all over the place.  You can also run into dependency and
versioning issues, that I don’t want to get into here, by placing libraries
here instead of keeping them in-tree or out-of-tree of the source code of a
project.  Just know when you see a library like `libc.so.6` that the numeric
suffix is a major version number that follows semantic versioning.  For more
information, you should read Michael Kerrisk’s excellent book *The Linux
Programming Interface*.  This post is based on his chapter’s 41 & 42 (but with
more info on tooling and OSX).

If we were to place our libhello.so into /usr/local/lib (on Linux you need to
then run `sudo ldconfig`) and move x.h and y.h to /usr/local/include, then we
could then compile with:

```sh
$ clang main.c -lhello
```

Note that rather than give a full path to our library, we can use the `-l` flag
followed by the name of our library with the lib prefix and .so suffix removed.

When working with shared libraries and external code, three flags I use pretty
often:

```
* -l<libname to link, no lib prefix or file extension; ex: -lnanomsg to link libnanomsg.so>
* -L <path to search for lib if in non standard directory>
* -I <path to headers for that library, if in non standard directory>
```

For finding specific flags needed for compilation where dynamic linkage is
required, a tool called `pkg-config` can be used for finding appropriate flags.
I’ve had less than stellar experiences with the tool as it puts the onus on the
library author to maintain the .pc files, and the user to have them installed
in the right place that `pkg-config` looks.  When they do exist and are
installed properly, the tool works well:

```sh
$ sudo apt-get install libpng12-dev
$ pkg-config --libs --cflags libpng12
-I/usr/include/libpng12  -lpng12
$ clang program.c `!!`
```

Using another neat environmental variable, we can hook into the dynamic linkage
process and inject our own shared libraries to be linked instead of the
expected libraries.  Let’s say libgood.so and libmalicous.so both define a
symbol for a function (the same symbol name and function signature).  We can
get a binary that links in libgood.so’s function to instead call
libmalicous.so’s version:


```sh
$ ./a.out
hello from libgood
$ LD_PRELOAD=./libmalicious.so ./a.out
hello from libmalicious
```

LD_PRELOAD is not available on OSX, instead you can use
`DYLD_INSERT_LIBRARIES`, `DYLD_LIBRARY_PATH`, and recompile the original
executable with `-flat_namespace`. Having to recompile the original executable
is less than ideal for hooking an existing binary, but I could not hook libc
as in the previous libmalicious example.  I would be interested to know if you
can though!

Manually invoking the dynamic linker from our code,
[we can even man in the middle library calls (call our hooked function first, then invoke the original target)](https://rafalcieslak.wordpress.com/2013/04/02/dynamic-linker-tricks-using-ld_preload-to-cheat-inject-features-and-investigate-programs/).
We’ll see more of this in the next post on using the dynamic linker.

As you can guess, readjusting the search paths for dynamic libraries is a
security concern as it let’s good and bad folks change the expected execution
paths.  Guarding against the use of these env vars becomes a rabbit hole that
gets pretty tricky to solve without the heavy handed use of statically linking
dependencies.

In the the previous post, I alluded to undefined symbols like `puts`.  `puts`
is part of libc, which is probably the most shared dynamic library on most
computing devices since most every program makes use of the C runtime.  (I
think of a “runtime” as implicit code that runs in your program that you didn’t
necessarily write yourself.  Usually a runtime is provided as a library that
gets implicitly linked into your executable when you compile.)  You can
statically link against libc with the `-static` flag, on Linux at least (OSX
makes this difficult,
["Apple does not support statically linked binaries on Mac OS X"](https://developer.apple.com/library/content/qa/qa1118/_index.html)).

I’m not sure what the benefit would be to mixing static and dynamic linking,
but after searching the search paths from LD_DEBUG=libs for shared versions of
a library, if any static ones are found, they will get linked in.

There’s also an interesting form of library called a "virtual dynamic shared
object" on Linux.  I haven’t covered memory mapping yet, but know it exists, is
usually hidden for libc, and that you can read more about it via `man 7 vdso`.

One thing I find interesting and don’t quite understand how to recreate is that
somehow glibc on Linux is also executable:

```sh
$ /lib/x86_64-linux-gnu/libc.so.6
GNU C Library (Ubuntu GLIBC 2.24-3ubuntu1) stable release version 2.24, by Roland McGrath et al.
Copyright (C) 2016 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.
There is NO warranty; not even for MERCHANTABILITY or FITNESS FOR A
PARTICULAR PURPOSE.
Compiled by GNU CC version 6.2.0 20161005.
Available extensions:
    crypt add-on version 2.1 by Michael Glad and others
    GNU Libidn by Simon Josefsson
    Native POSIX Threads Library by Ulrich Drepper et al
    BIND-8.2.3-T5B
libc ABIs: UNIQUE IFUNC
For bug reporting instructions, please see:
<https://bugs.launchpad.net/ubuntu/+source/glibc/+bugs>.
```

Also, note that linking against third party code has licensing implications (of
course) of particular interest when it’s GPL or LGPL.
[Here is a good overview](http://stackoverflow.com/a/10179181/1027966)
which I’d summarize as: code that *statically* links against LGPL code must
also be LGPL, while any form of linkage against GPL code must be GPL’d.

Ok, that was a lot. In the previous post, we covered
[Object Files and Symbols](/blog/2016/08/13/object-files-and-symbols/).
In this post we covered hacking around with static and dynamic linkage.  In the
next post, I hope to talk about manually invoking the dynamic linker at
runtime.

* [Part 1 - Object Files and Symbols](/blog/2016/08/13/object-files-and-symbols/)
* [Part 2 - Static and Dynamic Libraries](/blog/2016/11/20/static-and-dynamic-libraries/)
