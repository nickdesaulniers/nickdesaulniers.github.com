---
layout: post
title: "Speeding up Linux kernel builds with ccache"
date: 2018-06-02 16:39
comments: true
categories: Linux, kernel, C, ccache, compile
---
[ccache](https://ccache.samba.org/), the compiler cache, is a fantastic way to
speed up build times for C and C++ code that
[I previously recommended](https://nickdesaulniers.github.io/blog/2015/07/23/additional-c-slash-c-plus-plus-tooling/).
Recently, I was playing around with trying to get it to speed up my Linux
kernel builds, but wasn't seeing any benefit. Usually when this happens with
ccache, there's something non-deterministic about the builds that prevents
cache hits.

Turns out
[someone asked this exact question on the ccache mailing list back in 2014](https://lists.samba.org/archive/ccache/2014q1/001172.html),
and a teammate from my Android days supposed a timestamp was the culprit. That,
and
[this LKML post from the KBUILD maintainer in 2011 about determinism](https://lwn.net/Articles/437864/)
helped me find
[commit 87c94bfb8ad35 ("kbuild: override build timestamp & version")](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=87c94bfb8ad354fb43d2caf870d7ca0b3f98dab3) that introduced manually overriding part of the version string that
contains the build timestamp that can be seen from:

```sh
$ cat /proc/version
Linux version 4.15.0-13-generic (buildd@lgw01-amd64-028)
(gcc version 5.4.0 20160609 (Ubuntu 5.4.0-6ubuntu1~16.04.9))
#14~16.04.1-Ubuntu SMP
Sat Mar 17 03:04:59 UTC 2018
```

With ccache, we can check the cache hit/miss stats with `-s`, clear the cache
with `-C`, and clear the stats with `-z`. We can tell ccache explicitly which
compiler to fall back to as its first argument (not strictly necessary). For
KBUILD, we can swap our compiler by using `CC=` arg.

Let's see what happens to
our build time for subsequent builds with a hot cache:

### No Cache

```sh
$ make clean
$ time make -j4
...
make -j4  2008.93s user 231.69s system 346% cpu 10:47.07 total
```

### Cold Cache

```sh
$ ccache -Cz
Cleared cache
Statistics cleared
$ ccache -s
cache directory                     /home/nick/.ccache
primary config                      /home/nick/.ccache/ccache.conf
secondary config      (readonly)    /etc/ccache.conf
cache hit (direct)                     0
cache hit (preprocessed)               0
cache miss                             0
cache hit rate                      0.00 %
cleanups performed                     0
files in cache                         0
cache size                           0.0 kB
max cache size                       5.0 GB

$ make clean
$ time KBUILD_BUILD_TIMESTAMP='' make CC="ccache gcc" -j4
...
KBUILD_BUILD_TIMESTAMP='' make CC="ccache gcc" -j4  2426.79s user 312.08s system 372% cpu 12:15.22 total
$ ccache -s
cache directory                     /home/nick/.ccache
primary config                      /home/nick/.ccache/ccache.conf
secondary config      (readonly)    /etc/ccache.conf
cache hit (direct)                     0
cache hit (preprocessed)               0
cache miss                          3242
called for link                        6
called for preprocessing             538
unsupported source language           66
no input file                        108
files in cache                      9720
cache size                         432.6 MB
max cache size                       5.0 GB
```

### Hot Cache

```sh
$ ccache -z
Statistics cleared
$ make clean
$ time KBUILD_BUILD_TIMESTAMP='' make CC="ccache gcc" -j4
...
KBUILD_BUILD_TIMESTAMP='' make CC="ccache gcc" -j4  151.85s user 132.98s system 288% cpu 1:38.90 total
$ ccache -s
cache directory                     /home/nick/.ccache
primary config                      /home/nick/.ccache/ccache.conf
secondary config      (readonly)    /etc/ccache.conf
cache hit (direct)                  3232
cache hit (preprocessed)               7
cache miss                             3
called for link                        6
called for preprocessing             538
unsupported source language           66
no input file                        108
files in cache                      9734
cache size                         432.8 MB
max cache size                       5.0 GB
```

The initial cold cache build will be slower than not using ccache at all, but
it's a one time cost that's not significant relative to the savings.  No
caching took 647.07s, initial cold cache build took 735.22s (13.62% slower),
and subsequent hot cache builds took 98.9s (6.54x faster). YMMV based on CPU
and disk performance. Also, it's not the most common workflow to do clean
builds, but we do this for Linux kernel builds for Android/Pixel at work, and
this helps me significantly for local development.

Now, if you really need that date string in there, you *theoritically* could
put some garbage value in there (for the cache) long enough to save enough
space for a date string, then patch your vmlinux binary after the fact.  I
*don't* recommend that, but I would imagine that might look something like:

```sh
$ KBUILD_BUILD_TIMESTAMP=$(printf "y%0.s" $(seq 1 $(date | wc -c)))
$ make CC="ccache gcc" vmlinux
$ sed -i "s/$(KBUILD_BUILD_TIMESTAMP)/$(date)/g" vmlinux
```

Deterministic builds make build caching easier, but I'm not sure that build
timestamp strings and build determinism are reconcileable.
