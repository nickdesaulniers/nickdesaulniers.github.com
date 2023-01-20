---
title: "Debugging -Wframe-larger-than="
date: 2023-01-20T09:27:32-08:00
comments: true
categories:
- debugging
---
Unless work is done per architecture to implement
[HAVE_ARCH_VMAP_STACK](https://docs.kernel.org/mm/vmalloced-kernel-stacks.html)
/ `CONFIG_VMAP_STACK`,
the Linux kernel
[defaults to two pages worth of stack per thread](https://docs.kernel.org/x86/kernel-stacks.html).

---
Note: on many contemporary systems the page size is 4KiB, but this is actually
configurable for many architectures. The trade offs probably require a separate
post. If you see code that checks for alignment via bitwise tricks like `addr &
4095 == 0` without checking `sysconf(_SC_PAGESIZE)` it is perhaps a red flag
for code that might be to reused on different systems.

---

As a first line of defense against overflowing a thread's kernel stack, we
enable `-Wframe-larger-than=` with a value based on `CONFIG_FRAME_WARN`
(commonly `1024`). This doesn't guarantee we won't recurse enough at runtime to
overflow the stack. Defending against that is akin to solving the
[Halting Problem](https://en.wikipedia.org/wiki/Halting_problem)
unless you want to go to the extreme length of
[MISRA C's guidance](https://rules.sonarsource.com/c/RSPEC-925)
that "functions should not call themselves, either directly or indirectly."

It does help us
[frequently](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?qt=grep&q=-Wframe-larger-than)
find large structs that were stack allocated and probably should have been heap
(or sometimes statically) allocated. But the ergonomics of this warning have
room for improvement.

Let's say one day the build is failing because someone has introduced a new
instance of `-Wframe-larger-than=`. In the logs you see:
```
<source>:8:6: warning: stack frame size (4104) exceeds limit (1024) in 'foo' [-Wframe-larger-than]
void foo (void) {
     ^
```
So you go and look at source and see:
```c
void foo (void) {
    bar();
    baz();
    quux();
}
```
See any large local variables there? Thus begins the goose chase to understand
what inlining decisions led to `foo` having a large stack frame. Or how about
this?
```c
void baz (void) {
    struct widget;
    ...
```
is a `struct widget` too large to be putting on the stack? Let's look at the
definition:
```c
struct widget {
    struct gadget gadget;
    struct trombone trombone;
    long data [42];
    ...
```
What's the `sizeof` `struct widget`? Can you do that calculation in your head
quickly? What if the definitions of those structs are in other headers? More
goose chasing and perhaps an argument in favor of an IDE.

[DWARF](https://dwarfstd.org/) has this information (if/when
it's produced) but we don't have really great ways to visualize this
information. DWARF is a Jack of All Trades, it's good at many things, but kind
of great at none and so gets easily dunked on leading to distinct
[unwind formats](https://blogs.oracle.com/linux/post/unwinding-stack-frame-pointers-and-orc)
and
[type formats](https://docs.kernel.org/bpf/btf.html) being used in the kernel.

While commiserating about this conundrum with my colleague,
[Paul Kirth](https://github.com/ilovepi), I mentioned that I really wished
there was tooling that would "__just break out the crayons and draw me a picture
of the stack__ [usage of a given function]." Well, Paul must have been more upset
than I was, because he implemented some really nice optimization remarks to
help.

Starting with clang-16, you should be able to use
`-Rpass-analysis=stack-frame-layout` to get a drawing of your stack layout.

Let's see if we can better debug an instance of `-Wframe-larger-than=`
affecting the Linux kernel. Looking at
[logs from the CI from last night](https://github.com/ClangBuiltLinux/continuous-integration2/actions/runs/3967665297/jobs/6803182658#step:5:210),
I see a case:
```
Warning: /builds/linux/fs/jffs2/xattr.c:775:6: warning: stack frame size (1216) exceeds limit (1024) in 'jffs2_build_xattr_subsystem' [-Wframe-larger-than]
void jffs2_build_xattr_subsystem(struct jffs2_sb_info *c)
     ^
```
Looks like I can reproduce this locally:
```
$ wget https://src.fedoraproject.org/rpms/kernel/raw/rawhide/f/kernel-aarch64-fedora.config -O .config
$ make -s LLVM=1 ARCH=arm64 -j128 olddefconfig fs/jffs2/xattr.o
fs/jffs2/xattr.c:775:6: warning: stack frame size (1216) exceeds limit (1024) in 'jffs2_build_xattr_subsystem' [-Wframe-larger-than]
void jffs2_build_xattr_subsystem(struct jffs2_sb_info *c)
     ^
119/1216 (9.79%) spills, 1097/1216 (90.21%) variables
1 warning generated.
```

Paul also
[recently added that tidbit](https://reviews.llvm.org/rG2e1e2f52f357768186ecfcc5ac53d5fa53d1b094)
(that was cutoff from CI logs) about the relative number of spills vs
variables.  That can help you quickly get a sense how many stack slots are
variables' storage vs spills from excessive register pressure (too many live
values).

If we simply add `-Rpass-analysis=stack-frame-layout`, we're going to get a
beautiful ASCII table for every function in a given TU. We can use the flag
`-mllvm -fiter-print-funcs=` to reduce the number of optimization remarks
emitted. For the kernel, that might look like:
```
make -s LLVM=1 ARCH=arm64 -j128 fs/jffs2/xattr.o KCFLAGS="-Rpass-analysis=stack-frame-layout -mllvm -filter-print-funcs=jffs2_build_xattr_subsystem" 
fs/jffs2/xattr.c:775:6: warning: stack frame size (1216) exceeds limit (1024) in 'jffs2_build_xattr_subsystem' [-Wframe-larger-than]
void jffs2_build_xattr_subsystem(struct jffs2_sb_info *c)
     ^
119/1216 (9.79%) spills, 1097/1216 (90.21%) variables
fs/jffs2/xattr.c:776:1: remark: 
Function: jffs2_build_xattr_subsystem
Offset: [SP-8], Type: Spill, Align: 8, Size: 8
Offset: [SP-16], Type: Spill, Align: 8, Size: 8
Offset: [SP-24], Type: Spill, Align: 8, Size: 8
Offset: [SP-32], Type: Spill, Align: 8, Size: 8
Offset: [SP-40], Type: Spill, Align: 8, Size: 8
Offset: [SP-48], Type: Spill, Align: 8, Size: 8
Offset: [SP-56], Type: Spill, Align: 8, Size: 8
Offset: [SP-64], Type: Spill, Align: 8, Size: 8
Offset: [SP-72], Type: Spill, Align: 8, Size: 8
Offset: [SP-80], Type: Spill, Align: 8, Size: 8
Offset: [SP-88], Type: Spill, Align: 8, Size: 8
Offset: [SP-96], Type: Spill, Align: 8, Size: 8
Offset: [SP-104], Type: Variable, Align: 8, Size: 8
Offset: [SP-136], Type: Variable, Align: 8, Size: 28
    rr @ fs/jffs2/xattr.c:448
Offset: [SP-144], Type: Variable, Align: 8, Size: 8
Offset: [SP-1168], Type: Variable, Align: 8, Size: 1024
    xref_tmphash @ fs/jffs2/xattr.c:778
Offset: [SP-1176], Type: Spill, Align: 8, Size: 8
Offset: [SP-1180], Type: Spill, Align: 4, Size: 4
Offset: [SP-1192], Type: Spill, Align: 8, Size: 8
Offset: [SP-1196], Type: Spill, Align: 4, Size: 4 [-Rpass-analysis=stack-frame-layout]
{
^
1 warning generated.
```
(This kernel config had debug info enabled. Without this, lines above printing
variable and line number for `rr` and `xref_tmphash` would be omitted).

Spills can be occupied by different variables at different points for the
program counter (this is how DWARF encodes `DW_AT_location`). Not sure about
the Variable slots at Offsets `[SP-104]` and `[SP-144]` yet, maybe there's more
to fix in this nascent analysis, but the sizes show those aren't the droids...
err... stack slots that I'm looking for.

So right off the bat, if I'm setting `-Wframe-larger-than=1024` and
`xref_tmphash` is 1024B, that's a problem. `fs/jffs2/xattr.c:778`
[corresponds to this statement](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/fs/jffs2/xattr.c?id=edc00350d205d2de8871b514c8f9b403d588e5d1#n778).
```c
#define XREF_TMPHASH_SIZE	(128)
...
struct jffs2_xattr_ref *xref_tmphash[XREF_TMPHASH_SIZE];
```
It doesn't matter what the `sizeof` `struct jffs2_xattr_ref` since
`xref_tmphash` is a 128 element array of pointers to such struct. 128 times 8
(pointers are 64b on the kernel's aarch64 target) is 1024.

Let's see if anyone has tried to fix this. Yep, looking at [lore](https://lore.kernel.org/lkml/?q=jffs2_build_xattr_subsystem) I see:
- [fix sent in 2016](https://lore.kernel.org/lkml/1452193868-15816-1-git-send-email-tim.gardner@canonical.com/)
  - [rejected](https://lore.kernel.org/lkml/1454338130.133285.305.camel@infradead.org/)
- [another fix sent in 2017](https://lore.kernel.org/lkml/20170509203003.11986-1-fabf@skynet.be/) (looks like it never received feedback)
- [report from my co-maintainer](https://lore.kernel.org/lkml/YTbOs13waorzamZ6@Ryzen-9-3900X.localdomain/) [Nathan](https://nathanchance.dev/) in 2021
- report from [bot](https://lore.kernel.org/lkml/202110250146.2w7fdWCN-lkp@intel.com/) in 2021
- [report to stable last year](https://lore.kernel.org/lkml/78cae1c8-9a2b-b0a4-d9a1-efeb03290f58@w6rz.net/)

The patch from 2017 LGTM, and perhaps has fallen through the cracks and just
needs to be pinged/reviewed,
[which I've done](https://lore.kernel.org/lkml/Y8sN9+0BvZfsEq1v@google.com/).
Hopefully it can get picked up.

`-Wframe-larger-than=` issues in the Linux kernel are numerous (at least with
clang builds for now). They can be a bit of work to track down. One of
[our oldest open issues](https://github.com/ClangBuiltLinux/linux/issues/39)
still on the TODO list to fix is `-fsanitize=kernel-address` (via
`CONFIG_KASAN=y`) leading to excessive stack usage (at least when compared to
`-fsanitize=address` aka ASAN). Another
[TODO](https://github.com/llvm/llvm-project/issues/41896)
seems to be related to passing structs by value. My hope is these optimization
remarks will help us differentiate between compiler bugs vs kernel source bugs
quicker.

Whether or not an optimization remark is the most ergonomic tooling is
hopefully still up for debate, though the current implementation was an
iteration based on compromise. Requiring debug info for better diagnostics
trades compile time in clang, which is also unfortunate.

For now though, I'm happy to celebrate improved tooling
in this regard.  Cheers [Paul](https://github.com/ilovepi)!

That's pretty much the end of the post.  Some other tools I've used in this
area if you're still interested:

---
`llvm-dwarfdump` or GNU `objdump --dwarf=info` can print the DWARF stream. I
had written
[a python script](https://github.com/ClangBuiltLinux/frame-larger-than)
to attempt to decode this information. It's manually tested, incomplete, and
quite buggy. The library I depend on for decoding DWARF and ELF doesn't support
all of the architectures the Linux kernel does, which has led to issues being
unable to debug these warnings for specific architectures. A tool failure when
things are actively on fire is stressful.

```
$ frame_larger_than.py arch/x86/kernel/kvm.o kvm_send_ipi_mask_allbutself
kvm_send_ipi_mask_allbutself:
        1024    struct cpumask          new_mask
        4       unsigned int            this_cpu
        8       const struct cpumask*   local_mask
        4       int                     pscr_ret__
        4       int                     pfo_ret__
cpumask_copy:
bitmap_copy:
        4       unsigned int            len
        4       unsigned int            len
cpumask_clear_cpu:
clear_bit:
arch_clear_bit:
```

---
"Poke-a-hole" aka `pahole` can print the size of structs. Check out the [LWN
article](https://lwn.net/Articles/335942/).
```
# Make sure you've built with debug info enabled!
$ pahole fs/jffs2/xattr.o
...
struct jffs2_xattr_ref {
        void *                     always_null;          /*     0     8 */
        struct jffs2_raw_node_ref * node;                /*     8     8 */
        uint8_t                    class;                /*    16     1 */
        uint8_t                    flags;                /*    17     1 */
        u16                        unused;               /*    18     2 */
        uint32_t                   xseqno;               /*    20     4 */
        union {
                struct jffs2_inode_cache * ic;           /*    24     8 */
                uint32_t           ino;                  /*    24     4 */
        };                                               /*    24     8 */
        union {
                struct jffs2_inode_cache * ic;                   /*     0     8 */
                uint32_t                   ino;                  /*     0     4 */
        };

        union {
                struct jffs2_xattr_datum * xd;           /*    32     8 */
                uint32_t           xid;                  /*    32     4 */
        };                                               /*    32     8 */
        union {
                struct jffs2_xattr_datum * xd;                   /*     0     8 */
                uint32_t                   xid;                  /*     0     4 */
        };

        struct jffs2_xattr_ref *   next;                 /*    40     8 */

        /* size: 48, cachelines: 1, members: 9 */
        /* last cacheline: 48 bytes */
};
```

---
The kernel has a script (`scripts/stackusage`) that can print the estimated stack usage of each function in `vmlinux` to a file. Example:
```
$ ./scripts/stackusage LLVM=1 -j128 defconfig all
...
./scripts/stackusage: output written to /tmp/stackusage.3991505.3gRw
$ cat /tmp/stackusage.3991505.3gRw
arch/x86/entry/common.c:119:do_int80_syscall_32	24	dynamic
arch/x86/entry/common.c:138:__do_fast_syscall_32	40	dynamic
arch/x86/entry/common.c:186:do_fast_syscall_32	16	static
arch/x86/entry/common.c:238:do_SYSENTER_32	0	static
...
```

---
Finally, GCC currently supports (but clang currently does not)
`-fconserve-stack`. From playing with it, it seems that this flag causes GCC to
limit inlining if would increase the stack usage of a caller beyond what
appears to be through experimentation an arch specific threshold.

Clang [recently](https://github.com/llvm/llvm-project/commit/8564e2fea559c58fecab3c7c01acf498bbe7820a) got `-finline-max-stacksize=`, which feels like a nuclear option to
me.  We haven't deployed it yet in the kernel, but it might ultimately be
necessary to use. We'll see. I would hate to potentially cover up other issues
that should perhaps be fixed first.
