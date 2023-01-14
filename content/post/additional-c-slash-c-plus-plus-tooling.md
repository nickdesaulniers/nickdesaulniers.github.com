+++
title = "Additional C/C++ Tooling"
date = "2015-07-23"
slug = "2015/07/23/additional-c-slash-c-plus-plus-tooling"
Categories = ["C", "C++", "clang", "llvm", "toolchain"]
+++
[21st Century C by Ben Klemens](http://shop.oreilly.com/product/0636920025108.do)
was a great read. It had a section with an
intro to autotools, git, and gdb.
There are a few other useful tools that came to mind that I've used when
working with C and C++ codebases. These tools are a great way to start
contributing to
[Open Source](https://github.com/nickdesaulniers/What-Open-Source-Means-To-Me#what-open-source-means-to-me)
C & C++ codebases; running these tools on
the code or adding them to the codebases.  A lot of these favor command line,
open source utilities. See how many you are familiar with!

## Build Tools
### CMake
The first tool I'd like to take a look at is
[CMake](http://www.cmake.org/overview/).  CMake is yet another build tool; I
realize how contentious it is to even discuss one of the many.  From my
experience working with
[Emscripten](https://kripken.github.io/emscripten-site/docs/introducing_emscripten/about_emscripten.html),
we recommend the use of CMake for people
writing portable C/C++ programs.  CMake is able to emit Makefiles for unixes,
project files for Xcode on OSX, and project files for Visual Studio on Windows.
There are also a few other "generators" that you can use.

I've been really impressed with CMake's modules for
[finding dependencies](http://www.cmake.org/cmake/help/v3.0/command/find_package.html)
and
[another for fetching and building external dependencies](http://www.cmake.org/cmake/help/v3.0/module/ExternalProject.html).
I think
[C++ needs a package manager badly](https://www.youtube.com/watch?v=nshzjMDD79w),
and I think CMake would be a solid foundation for one.

The syntax isn't the greatest, but when I wanted to try to build one of my C++
projects on Windows which I know nothing about developing on, I was able to
install CMake and Visual Studio and get my project building.  If you can build
your code on one platform, it will usually build on the others.

If you're not worried about writing cross platform C/C++, maybe CMake is not
worth the effort, but I find it useful.  I wrestle with the syntax sometimes,
but documentation is not bad and it's something you deal with early on in the
development of a project and hopefully never have to touch again (how I wish
that were true).

## Code Formatters
### ClangFormat
Another contentious point of concern amongst developers is code style.
[Big companies](http://google-styleguide.googlecode.com/svn/trunk/cppguide.html)
with lots of C++ code have
[documents](https://developer.mozilla.org/en-US/docs/Mozilla/Developer_guide/Coding_Style#CC_practices)
explaining their stylistic choices.  Don't waste another hour of your life
arguing about something that really doesn't matter.
[ClangFormat](http://clang.llvm.org/docs/ClangFormat.html) will help you
codify your style and format your code for you to match the style.  Simply
write the code however you want, and run the formatter on it before commiting
it.

It can also emit a .clang-format file that you can commit and clang-format will automatically look for that file and use the rules codified there.

## Linters
### Flint / Flint++
[Flint](https://github.com/facebook/flint) is a C++ linter in use at Facebook.
Since it moved from being
implemented in C++ to D, I've had issues building it.  I've had better luck
with a fork that's pure C++ without any of the third party dependencies Flint
originally had, called
[Flint++](https://github.com/L2Program/FlintPlusPlus).  While not quite full-on
static analyzers, both can be used for finding potential issues in your code
ahead of time. Linters can look at individual files in isolation; you don't
have to wait for long recompiles like you would with a static analyzer.

## Static Analyzers
### Scan-build
[Scan-build](http://clang-analyzer.llvm.org/scan-build.html) is a static
analyzer for C and C++ code.  You build your code "through" it, then use the
sibling tool scan-view to see the results.  Scan-view will emit and open an
html file that shows a list of the errors it detected.  It will insert
hyperlinks into the resulting document that step you through how certain
conditions could lead to a null pointer dereference, for example.  You can also
save and share those html files with others in the project. Static analyzers
will help you catch bugs at compile time before you run the code.

## Runtime Sanitizers
### ASan and UBSan
Clang's Address (ASan) and Undefined Behavior (UBSan) sanitizers are simply
compiler flags that can be used to detect errors at runtime.  ASan and UBSan
two of the more popular tools, but there are actually a ton and more being
implemented.  See the list
[here](http://clang.llvm.org/docs/UsersManual.html#controlling-code-generation).
These sanitizers will catch bugs at runtime, so you'll have to run the code
to notice any violations, at variable runtime performance costs per sanitizer.
ASan and TSan (Thread Sanitizer) made it into gcc4.8 and UBSan is in gcc4.9.

## Header Analysis
### Include What You Use
[Include What You Use](https://github.com/include-what-you-use/include-what-you-use)
(IWYU) helps you find unused or unnecessary `#include` preprocessor directives.
It should be obvious how this can help improve compile times. IWYU can also
help cut down on recompiles by recommending forward declarations under certain
conditions.
I look forward to the C++ module proposal being adopted, but until then this
tool can help you spot cruft that can be removed.

## Rapid Recompiles
### ccache
[ccache](https://ccache.samba.org/) greatly improves recompile times by caching
the results of parts of the compilation process.
[I use when building Firefox](https://github.com/nickdesaulniers/dotfiles/blob/49984b3e82022e5ce82e778fc8ce990f8e1e554a/.mozconfig#L1),
and it saves a great deal of time.

### distcc
[distcc](https://github.com/distcc/distcc) is a distributed build system.
[Some folks at Mozilla](http://blog.dholbert.org/) speed up their Firefox builds with it.

## Memory Leak Detectors
### Valgrind
[Valgrind](http://valgrind.org/info/about.html) has a
[suite of tools](http://valgrind.org/info/about.html), my
favorite being memcheck for finding memory leaks. Unfortunately, it doesn't
seem to work on OSX since 10.10.
[This page](https://code.google.com/p/address-sanitizer/wiki/ComparisonOfMemoryTools)
referring to ASan seems to indicate that it can do everything Valgrind's
Memcheck can, at less of a runtime performance cost, but I'm not sure how true
this is exactly.

### leaks
A much more primitive tool for finding leaks from the command line, BSD's have
`leaks`.

```bash
MallocStackLogging=1 ./a.out
leaks a.out
...
```

## Profilers
### Perf
Perf, and
[Brendan Gregg's tools for emitting SVG flamegraphs](http://www.brendangregg.com/flamegraphs.html)
from the output
are helpful for finding where time is spent in a program. In fact, there are
numerous perfomance analysis tools that are Linux specific.  My recommendation
is spend some time on [Brendan Gregg's blog](http://www.brendangregg.com/linuxperf.html).

### DTrace
OSX doesn't have the same tooling as Linux, but DTrace was ported to it.  I've
used it to find sampling profiles of my code before. Again,
[Brendan Gregg's blog](http://www.brendangregg.com/dtrace.html) is a good
resource; there are some fantastic DTrace one liners.

## Debuggers
### lldb
lldb is analogous to gdb.  I can't say I have enough experience with LLDB and GDB to note the difference between the two, but LLDB did show the relative statements forward and back from the current statement by default.  I'm informed by my friends/mortal enemies using emacs that this is less of an issue when using emacs/gdb in combination.

## Fuzzers
### American Fuzzy Lop
[American Fuzzy Lop](http://lcamtuf.coredump.cx/afl/) (AFL) is a neat program
that performs fuzzing on programs
that take inputs from files and repeatedly runs the program, modifies the
input trying to get full code coverage, and tries to find crashes.  It's been
getting lots of attention lately, and while I haven't used it myself yet, it
seems like a very powerful tool. Mozilla employs the use of fuzzers on their
JavaScript engine, for instance (not AFL, but
[one developed in house](http://www.squarefree.com/2007/08/02/introducing-jsfunfuzz/)).

## Disassemblers
### gobjdump
If you really need to make sure the higher level code you're writing is getting
translated into the assembly your expecting, `gobjdump -S` will intermix the
emitted binary's disassembled assembly and the source code.  This was used
extensively while developing [my Brainfuck JIT](/blog/2015/05/25/interpreter-compiler-jit/).

## Conclusion
Hopefully you learned of some useful tools that you should know about when
working with C or C++.  What did I miss?

