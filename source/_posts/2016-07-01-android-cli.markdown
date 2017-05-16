---
layout: post
title: "Cross Compiling C/C++ for Android"
date: 2016-07-01 22:42
comments: true
categories: Android NDK C C++ ARM ARMv8 ARMv8-A AArch64 Compile Standalone CLI Command Line Toolchain Nexus Cross
---
Let’s say you want to build a hello world command line application in C or C++
and run it on your Android phone.  How would you go about it?  It’s not super
practical; apps visible and distributable to end users must use the framework
(AFAIK), but for folks looking to get into developing on ARM it’s likely they
have an ARM device in their pocket.

This post is for folks who typically invoke their compiler from the command
line, either explicitly, from build scripts, or other forms of automation.

At
[work](https://twitter.com/LostOracle/status/697859368226697218),
when working on Android, we typically checkout the entire Android source code
([which is huge](https://twitter.com/LostOracle/status/702569487531249664)),
use `lunch` to configure a ton of environmental variables, then use Makefiles
with lots of includes and special vars.  We don’t want to spend the time and
disk space checking out the Android source code just to have a working cross
compiler.  Luckily, the Android tools team has an excellent utility to grab a
prebuilt cross compiler.

This assumes you’re building from a Linux host.  Android is a distribution of
Linux, which is much easier to target from a Linux host.  At home, I’ll usually
develop on my OSX machine, ssh’d into my headless Linux box. (iTerm2 and tmux
both have exclusive features, but I currently prefer iTerm2.)

The first thing we want to do is fetch the
[Android NDK](https://developer.android.com/ndk/downloads/index.html).
Not the SDK, the NDK.

```sh
➜  ~ curl -O \
  http://dl.google.com/android/repository/android-ndk-r12b-linux-x86_64.zip
➜  ~ unzip android-ndk-r12b-linux-x86_64.zip
```

It would be helpful to install adb and fastboot, too.  This might be different
for your distro’s package manager.  Better yet may be to just build from
source.

```sh
➜  ~ sudo apt-get install android-tools-adb android-tools-fastboot
```

Now for you Android device that you want to target, you’ll want to know the
ISA.  Let’s say I want to target my Nexus 6P, which has an ARMv8-A ISA (the
first 64b ARM ISA).

```sh
➜  ~ ./android-ndk-r12b/build/tools/make_standalone_toolchain.py --arch arm64 \
  --install-dir ~/arm
```

This will create a nice standalone bundle in `~/arm`.  It will contain our
cross compiler, linker, headers, libs, and
[sysroot (crt.o and friends)](https://twitter.com/LostOracle/status/749297676223598592).
Most Android devices are ARMv7-A, so you’d use `--arch arm`.  See the other
supported architectures for cross compiling under
[table 4](https://developer.android.com/ndk/guides/standalone_toolchain.html#itc).


You might also want to change your install-dir and possible add it to your
`$PATH`, or set `$CC` and `$CXX`.

Now we can compile `hello_world.c`.

```sh
➜  ~ cat hello_world.c
#include <stdio.h>
int main () {
  puts("hello world");
}

➜  ~ ~/arm/bin/clang -pie hello_world.c
➜  ~ file a.out
a.out: ELF 64-bit LSB shared object, ARM aarch64, version 1 (SYSV), dynamically
linked, interpreter /system/bin/linker64,
BuildID[sha1]=ecc46648cf2c873253b3b522c0d14b91cf17c70f, not stripped
```

[Since Android Lollipop](http://stackoverflow.com/a/30547603),
Android has required that executables be linked as
position independent (`-pie`) to help provide
[ASLR](https://en.wikipedia.org/wiki/Address_space_layout_randomization#Android).


`<install-dir>/bin/` also has shell scripts with more full featured names like
`aarch64-linux-android-clang` if you prefer to have clearer named executables
in your $PATH.

Connect your phone, enable remote debugging, and accept the prompt for remote
debugging.

```sh
➜  ~ adb push a.out /data/local/tmp/.
➜  ~ adb shell "./data/local/tmp/a.out"
hello world
```

We’ll use this toolchain in a follow post to start writing some ARMv8-A
assembly.  Stay tuned.
