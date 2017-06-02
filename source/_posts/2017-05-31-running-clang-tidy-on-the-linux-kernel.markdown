---
layout: post
title: "Running Clang-Tidy on the Linux Kernel"
date: 2017-05-31 20:25
comments: true
categories: clang tidy llvm linux kernel static analysis
---
[Clang-Tidy](http://clang.llvm.org/extra/clang-tidy/) is a linter from the LLVM
ecosystem.  I wanted to try to run it on the Linux kernel to see what kind of
bugs it would find.  The false positive rate seems pretty high (a persistent
bane to static analysis), but some patching in both the tooling and the source
can likely help bring this rate down.

The most straightforward way to invoke Clang-Tidy is with a compilation
database, which is a json based file that for each translation unit records

1. The source file of the translation unit.
2. The top level directory of the source.
3. The exact arguments passed to the compiler.

The exact arguments are required because `-D` and `-I` flags are necessary to
reproduce the exact Abstract Syntax Tree (AST) used to compile your code. Given
a compilation database, it's trivial to parse and recreate a build.  For the
kernel's KBuild, it's a lot like encoding the output of `make V=1`.

In order to generate a compilation database, we can use an awesome tool called
[BEAR](https://github.com/rizsotto/Bear). BEAR will
[hook](https://github.com/rizsotto/Bear/blob/6b07f5044f30a3070d1dc39801bcdd94395d673e/libear/ear.c#L21)
calls to
[exec](https://linux.die.net/man/3/exec)
and family, then write out the compilation database (compile_commands.json).

With BEAR installed, we can invoke the kernel's build with `bear make -j`. When
we're done:

```sh
➜  linux git:(nick) ✗ du -h compile_commands.json
11M compile_commands.json
➜  linux git:(nick) ✗ wc -l compile_commands.json
330296 compile_commands.json
➜  linux git:(nick) ✗ head -n 26 compile_commands.json
[
    {
        "arguments": [
            "cc",
            "-c",
            "-Wp,-MD,arch/x86/boot/tools/.build.d",
            "-Wall",
            "-Wmissing-prototypes",
            "-Wstrict-prototypes",
            "-O2",
            "-fomit-frame-pointer",
            "-std=gnu89",
            "-Wno-unused-value",
            "-Wno-unused-parameter",
            "-Wno-missing-field-initializers",
            "-I./tools/include",
            "-include",
            "include/generated/autoconf.h",
            "-D__EXPORTED_HEADERS__",
            "-o",
            "arch/x86/boot/tools/build",
            "arch/x86/boot/tools/build.c"
        ],
        "directory": "/home/nick/linux",
        "file": "arch/x86/boot/tools/build.c"
    },
```

Now with Clang-Tidy (probably worthwhile to build from source, but it's also
available off `apt`), we want to grab
[this helper script, run-clang-tidy.py](https://github.com/llvm-mirror/clang-tools-extra/blob/master/clang-tidy/tool/run-clang-tidy.py)
to help analyze all this code.

```sh
curl -O https://raw.githubusercontent.com/llvm-mirror/clang-tools-extra/master/clang-tidy/tool/run-clang-tidy.py
```

Then we can run it from the same directory as compile_commands.json:

```sh
python run-clang-tidy.py \
  -clang-tidy-binary /usr/bin/clang-tidy-4.0 \
  > clang_tidy_output.txt
```

This took about 1hr12min on my box. Let's see what the damage is:

```sh
➜  linux git:(nick) ✗ cat clang_tidy_output.txt \
  | grep warning: | grep -oE '[^ ]+$' | sort | uniq -c

     76 [clang-analyzer-core.CallAndMessage]
     15 [clang-analyzer-core.DivideZero]
      1 [clang-analyzer-core.NonNullParamChecker]
    316 [clang-analyzer-core.NullDereference]
     90 [clang-analyzer-core.UndefinedBinaryOperatorResult]
      1 [clang-analyzer-core.uninitialized.ArraySubscript]
   1410 [clang-analyzer-core.uninitialized.Assign]
     10 [clang-analyzer-core.uninitialized.Branch]
      5 [clang-analyzer-core.uninitialized.UndefReturn]
     11 [clang-analyzer-cplusplus.NewDeleteLeaks]
    694 [clang-analyzer-deadcode.DeadStores]
    342 [clang-analyzer-security.insecureAPI.strcpy]
      2 [clang-analyzer-unix.API]
     11 [clang-analyzer-unix.Malloc]
      4 [clang-diagnostic-address-of-packed-member]
      2 [clang-diagnostic-duplicate-decl-specifier]
     98 [clang-diagnostic-implicit-int]
```

Looking through the output, there's seems to be almost nothing but false
positives, but who knows, maybe there's an actual bug or two in there.  Likely
possible patches to LLVM, its checkers, or the Linux kernel could lower the
false positive ratio.

If you're interested in seeing the kinds of warnings/outputs, I've uploaded my
results run on a 4.12-rc3 based kernel that may or may not have been compiled
with Clang to
[my clang_tidy branch of the kernel on GitHub](https://github.com/nickdesaulniers/linux/blob/clang_tidy/clang_tidy_output.txt.v2).
As in my sorted output, I find it handy to `grep` for `warning:`. Maybe you can
find yourself a good first bug to
[contribute a fix to the kernel](blog/2017/05/16/submitting-your-first-patch-to-the-linux-kernel-and-responding-to-feedback/)?

There's likely also
[some checks that make sense to disable or enable](http://clang.llvm.org/extra/clang-tidy/checks/list.html).
Clang-Tidy also allows you to
[write and use your own checkers](http://clang.llvm.org/extra/clang-tidy/#writing-a-clang-tidy-check).
Who knows, someone may just end up writing static
analyses tailored to the Linux kernel.
