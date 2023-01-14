+++
title = "Finding Compiler Bugs With C Reduce"
date = "2019-01-18"
slug = "2019/01/18/finding-compiler-bugs-with-c-reduce"
Categories = ["C", "C++", "debugging", "c-reduce", "linux", "llvm"]
+++
Support for a long awaited GNU C extension,
[asm goto](https://gcc.gnu.org/onlinedocs/gcc/Extended-Asm.html),
is in the midst of landing in
[Clang](https://reviews.llvm.org/D56571) and
[LLVM](https://reviews.llvm.org/D53765).  We want to make sure that
we release a high quality implementation, so it's important to test the new
patches on real code and not just small test cases.  When we hit compiler bugs
in large source files, it can be tricky to find exactly what part of
potentially large translation units are problematic.  In this post, we'll take
a look at using
[C-Reduce](https://embed.cs.utah.edu/creduce/),
a multithreaded code bisection utility for C/C++, to help narrow done a
reproducer for
[a real compiler bug](https://github.com/ClangBuiltLinux/linux/issues/320)
(potentially; in a patch that was posted, and will be fixed before it can ship
in production) from a real code base (the Linux kernel).  It's mostly a post to
myself in the future, so that I can remind myself how to run C-reduce on the
Linux kernel again, since this is now the third real compiler bug it's helped
me track down.

So the bug I'm focusing on when trying to compile the Linux kernel with Clang
is a linkage error, all the way at the end of the build.
```
drivers/spi/spidev.o:(__jump_table+0x74): undefined reference to `.Ltmp4'
```
Hmm...looks like the object file (`drivers/spi/spidev.o`), has a
section (`__jump_table`), that references a non-existent
symbol (`.Ltmp`), which looks like a temporary label that should have been
cleaned up by the compiler.  Maybe it was accidentally left behind by an
optimization pass?

To run C-reduce, we need a shell script that returns 0 when it should keep
reducing, and an input file.  For an input file, it's just way simpler to
preprocess it; this helps cut down on the compiler flags that typically
requires paths (`-I`, `-L`).

## Preprocess

First, let's preprocess the source.  For the kernel, if the file compiles
correctly, the kernel's KBuild build process will create a file named in the
form path/to/.file.o.cmd, in our case drivers/spi/.spidev.o.cmd.  (If the file
doesn't compile, then
[I've had success](https://nickdesaulniers.github.io/blog/2017/05/31/running-clang-tidy-on-the-linux-kernel/)
hooking `make path/to/file.o` with
[bear](https://github.com/rizsotto/Bear)
then getting the `compile_commands.json` for the file.)  I find it easiest to
copy this file to a new shell script, then strip out everything but the first
line.  I then replace the `-c -o <output>.o` with `-E`.  `chmod +x` that new
shell script, then run it (outputting to stdout) to eyeball that it looks
preprocessed, then redirect the output to a `.i` file.  Now that we have our
preprocessed input, let's create the C-reduce shell script.

## Reproducer

I find it helpful to have a shell script in the form:

1. remove previous object files
2. rebuild object files
3. disassemble object files and pipe to grep

For you, it might be some different steps.
[As the docs show](https://embed.cs.utah.edu/creduce/using/),
you just need the shell script to return 0 when it should keep reducing.  From
our previous shell script that pre-processed the source and dumped a `.i` file,
let's change it back to stop before linking rather that preprocessing
(`s/-E/-c/`), and change the input to our new `.i` file.  Finally, let's add
the test for what we want.  Since I want C-Reduce to keep reducing until the
disassmbled object file no longer references anything `Ltmp` related, I write:

```sh
$ objdump -Dr -j __jump_table spidev.o | grep Ltmp > /dev/null
```

Now I can run the reproducer to check that it at least returns 0, which
C-Reduce needs to get started:

```sh
$ ./spidev_asm_goto.sh
$ echo $?
0
```

## Running C-Reduce

Now that we have a reproducer script and input file, let's run C-Reduce.

```
$ time creduce --n 40 spidev_asm_goto.sh spidev.i
===< 144926 >===
running 40 interestingness tests in parallel
===< pass_includes :: 0 >===
===< pass_unifdef :: 0 >===
===< pass_comments :: 0 >===
===< pass_blank :: 0 >===
(0.7 %, 2393679 bytes)
(5.3 %, 2282207 bytes)
===< pass_clang_binsrch :: replace-function-def-with-decl >===
(12.6 %, 2107372 bytes)
...
===< pass_indent :: final >===
(100.0 %, 156 bytes)
===================== done ====================

pass statistics:
  method pass_clang_binsrch :: remove-unused-function worked 1 times and failed 0 times
...
  method pass_lines :: 0 worked 427 times and failed 998 times
            ******** /android0/kernel-all/spidev.i ********

a() {
  int b;
  c();
  if (c < 2)
    b = d();
  else {
    asm goto("1:.long b - ., %l[l_yes] - . \n\t" : : : : l_yes);
  l_yes:;
  }
  if (b)
    e();
}
creduce --n 40 spidev_asm_goto.sh spidev.i  1892.35s user 1186.10s system 817% cpu 6:16.76 total
$ wc -l spidev.i.orig
56160 spidev.i.orig
$ wc -l spidev.i
12 spidev.i
```

So it took C-reduce just over 6 minutes to turn >56k lines of mostly irrelevant
code into 12 when running 40 threads on my 48 core workstation.

It's also highly entertaining to watch C-Reduce work its magic. In another
terminal, I highly recommend running `watch -n1 cat <input_file_to_creduce.i>`
to see it pared down before your eyes.

Jump to 4:24 to see where things really pick up.
[![asciicast](https://asciinema.org/a/XtD0QdiIUGhvc1G2BqTJ9gti2.svg)](https://asciinema.org/a/XtD0QdiIUGhvc1G2BqTJ9gti2)
[![asciicast](https://asciinema.org/a/zdkbvUqDsilSa5QjGJr3ANP6y.svg)](https://asciinema.org/a/zdkbvUqDsilSa5QjGJr3ANP6y)

Finally, we still want to bisect our compiler flags (the kernel uses a lot).  I
still do this process manually, and it's not too bad.  Having proper and
minimal steps to reproduce compiler bugs is critical.

That's enough for a great bug report for now.  In a future episode, we'll see
how to start pulling apart llvm to see where compilation is going amiss.
