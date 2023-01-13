+++
title = "Gcc vs Llvm Q3 2017 Commit Rates and Active Developer Counts"
date = "2017-09-05"
slug = "2017/09/05/gcc-vs-llvm-q3-2017-commit-rates-and-active-developer-counts"
Categories = []
+++
A blog post from a few years ago that really stuck with me was Martin Olsson’s
[Browser Engines 2015: Commit Rates and Active Developer Counts](https://mo.github.io/2015/11/04/browser-engines-active-developers-and-commit-rates.html),
where he shows information about the number of authors and commits to popular
web browsers.  The graphs and analysis had interesting takeaways like showing
the obvious split in blink and webkit, and relative number of contributors of
the projects.  Martin had data comparing gcc to llvm from Q4 2015, but I wanted
to see what the data looked like now in Q3 2017 and wanted to share my
findings; simply rerunning the numbers.  Luckily Martin
[open sourced](https://github.com/mo/git-source-metrics)
the scripts he used for measurements so they could be rerun.

Commit count and active authors in the previous 60 days is a rough estimate for
project health; the scripts don’t/can’t account for unique authors (same author
using different git commit info) and commit frequency is meaningless for
comparing developers that commit early and commit often, but let’s take a look.
Active contributors over 60 days cuts out folks who do commit to either code
bases, just not as often.  Lies, damn lies, and statistics, right? Or torture
the data enough, and it will confess anything...

Note that LLVM is split into a few repositories (llvm the common base, clang
the C/C++ frontend, libc++ the C++ runtime, compiler-rt the
sanitizers/built-ins/profiler lib, lld the linker, clang-tools-extra the
utility belt, lldb the debugger (there are more, these are the most active LLVM
projects)).  Later, I refer to LLVM as the grouping of these repos.

There’s a lot of caveats with this data.  I suspect that the separate LLVM
repo’s have a lot of overlap and have fewer active contributors when looked at
in aggregate.  That is to say you can’t simply add them else you’d be double
counting a bunch.  Also, the comparison is not quite fair since the overlap in
front-end-language and back-end-target support in these two massive projects
does not overlap in a lot of places.

![gcc clang authors](/images/gcc_clang_authors.jpg)

LLVM’s 60 day active contributors are ~3x-5x times GCC’s and growing, while
GCC’s 100-count hasn’t changed much since ‘04.  It’s safe to say GCC is not
dying; it’s going steady and chugging away as it has been, but it seems LLVM
has been very strong in attracting active contributors.  Either way, I’m
thankful to have not one, but two high quality open source C/C++ compilers.
