+++
title = "Off by Two"
date = "2020-04-06"
slug = "2020/04/06/off-by-two"
Categories = []
+++
"War stories" in programming are entertaining tales of truly evil bugs that
kept you up at night.  Inspired by posts like
[My Hardest Bug Ever](https://www.gamasutra.com/blogs/DaveBaggett/20131031/203788/My_Hardest_Bug_Ever.php),
[Debugging an evil Go runtime bug](https://marcan.st/2017/12/debugging-an-evil-go-runtime-bug/),
and others from
[/r/TalesFromDebugging](https://www.reddit.com/r/TalesFromDebugging), I wanted
to share with you one of my favorites from recent memory.
[Recent work](https://clangbuiltlinux.github.io/)
has given me much fulfilment and a long list of truly awful bugs to recount.
My blog has been quieter than I would have liked; hopefully I can find more
time to document some of these, maybe in series form.  May I present to you
episode I; "*Off by Two*."

---

Distracted in a conference grand ballroom, above what might be the largest mall
in the world or at least Bangkok, a blank QEMU session has me seriously
questioning my life choices.  No output.  Fuck!  My freshly built Linux kernel,
built with a large new compiler feature that’s been in development for months
is finally now building but is not booting.  Usually a panic prints a nice
stack trace and we work backwards from there.  I don’t know how to debug a
panic during early boot, and I’ve never had to; with everything I’ve learned up
to this point, I’m afraid I won’t have it in me to debug this.

[Attaching GDB](https://nickdesaulniers.github.io/blog/2018/10/24/booting-a-custom-linux-kernel-in-qemu-and-debugging-it-with-gdb/), the kernel’s sitting an infinite loop:

```c
/* Restricted version used during very early boot */
void __init early_fixup_exception(struct pt_regs *regs, int trapnr)
{
...
halt_loop:
	while (true)
		halt();
```

Some sort of very early exception handler; better to sit busy in an infinite
loop than run off and destroy hardware or corrupt data, I suppose.  It seems
this is some sort of exception handler for before we’re ready to properly
panic; maybe the machinery is not in place to even collect a stack trace,
unwind, and print that over the serial driver.  How did things go so wrong and
how did we get here?  I decide to ask for help.

> Setting breakpoints and rerunning my boot, it looks like the fourth
> call to __early_make_pgtable() is deterministically going awry.
> Reading through callers, from the early_idt_handler_common subroutine
> in arch/x86/kernel/head_64.S the address was stored in %cr2 (the
> "page fault linear address").  But it’s not clear to me who calculated
> that address that created the fault.  My understanding is that
> early_idt_handler_common is an exception vector setup in
> early_idt_handler_array, which gets invoked upon access to "unmapped
> memory" which gets saved into %cr2.

Beyond that, GDB doesn’t want me to be able to read %cr2.

[Jann Horn](https://thejh.net/) gets back to me first:

> Can you use QEMU to look at the hardware frame (which contains values pushed
> by the hardware in response to the page fault) in early_idt_handler_common?
> RSP before the call to early_make_pgtable should basically point to a "struct
> pt_regs"
>
> When the CPU encounters an exception, it pushes an exception frame onto the
> stack. That doesn't happen in kernel code; the CPU does that on its own. That
> exception frame consists of the last six elements of struct pt_regs. This is
> also documented in a comment at the start of early_idt_handler_common
> ("hardware frame" and "error code" together are the exception frame):
>
>     /*
>      * The stack is the hardware frame, an error code or zero, and the
>      * vector number.
>      */
>
> After the CPU has pushed that stuff, it picks one of the exception handlers
> that have been set up in idt_setup_early_handler(); so it jumps to
> &early_idt_handler_array[i]. early_idt_handler_array pushes the number of the
> interrupt vector, then calls into early_idt_handler_common;
> early_idt_handler_common spills the rest of the register state (which is
> still the way it was before the exception was triggered) onto the stack
> (which among other things involves reading the vector number into a register
> and overwriting the stack slot of the vector number with a register that
> hasn't been spilled yet).
>
> The combination of the registers that have been spilled by software and the
> values that have been pushed onto the stack by the CPU before that forms a
> struct pt_regs. (The normal syscall entry slowpath does the same thing, by
> the way.)
>
> you’ll want to break on the "call early_make_pgtable" or something like that,
> to get the pt_regs to be fully populated and at RSP.

This is documented further in
[linux-insides](https://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-2.html).

So as far as "where does the address in %cr2 come from, it’s "the CPU." To get
%cr2, I can just break after an instruction that moves %cr2 into a general
purpose register (GPR).

```c
	GET_CR2_INTO(%rdi)	/* can clobber %rax if pv */
	call early_make_pgtable

```

```sh
$ gdb -batch -ex "file vmlinux" -ex "disassemble
early_idt_handler_common" | grep early_make_pgtable
   0xffffffff82965150 <+48>: callq  0xffffffff829653b9 <early_make_pgtable>
$ gdb vmlinux
...
(gdb) hbreak *0xffffffff82965150
...
(gdb) p/x *(struct pt_regs*)0xffffffff82403e28
$1 = {r15 = 0x0, r14 = 0xffffffff82aa3808, r13 = 0x0, r12 = 0x0, bp = 0x0,
  bx = 0xfffffffc, r11 = 0x2794c5, r10 = 0x20, r9 = 0x13ca62, r8 =
0x20, ax = 0x0,
  cx = 0xffffffff82415700, dx = 0xfffffffb6df881bb, si = 0x0,
  di = 0xffffffff82aa3808, orig_ax = 0x0, ip = 0xffffffff81172b70, cs = 0x10,
  flags = 0x10006, sp = 0xffffffff82403ed8, ss = 0x0}
(gdb) x 0xffffffff81172b70
   0xffffffff81172b70 <jump_label_update+64>: mov    0x8(%rbx),%rsi
(gdb) q
$ objdump -dS vmlinux | grep -B 2 ffffffff81172b70
ffffffff81172b69: 0f 1f 80 00 00 00 00 nopl   0x0(%rax)
if (!mod->entries)
ffffffff81172b70: 48 8b 73 08          mov    0x8(%rbx),%rsi
```

Specifically `mod` in the above expression (ie. `rbx`) is not pointing to valid
memory in the page tables, triggering an unrecoverable early page fault.

My heart sinks further at the sight of `jump_lable_update`.  It’s `asm goto`,
the large compiler feature we’ve been working on for months, and it’s subtly
broken.  Welcome to hell, kids.

`asm goto` is a GNU C extension that allows for assembly code to transfer
control flow to a limited, known set of labels in C code.  Typically, regular
[`asm` statements](https://gcc.gnu.org/onlinedocs/gcc/Basic-Asm.html#Basic-Asm)
(the GNU C extension) are treated as a black box in the instruction stream by
the compiler; they’re called into (not in the sense of the C calling convention
and actual call/jmp/ret instructions) and control flow falls through to the
next instruction outside of the inline assembly.  Then there’s an
"[extended inline assembly](https://gcc.gnu.org/onlinedocs/gcc/Extended-Asm.html#Extended-Asm)"
dialect that allows for you to specify input and output constraints (in what
feels like a whole new regex-like language with characters that have
[architecture specific](https://gcc.gnu.org/onlinedocs/gcc/Machine-Constraints.html#Machine-Constraints)
or
[generic](https://gcc.gnu.org/onlinedocs/gcc/Simple-Constraints.html#Simple-Constraints)
meanings, and requires the reference manual to read or write) and whether to
treat all memory or specific registers otherwise unnamed as outputs as
clobbered.  In the final variant, you may also specify a list of labels that
the assembly may jump control flow to.  There's also `printf`-like modifiers
called
[Output Templates](https://gcc.gnu.org/onlinedocs/gccint/Output-Template.html#Output-Template),
and a few other tricks that require their own post.

Within the compiler, we can’t really treat `asm` statements like a black box
anymore.  With `asm goto`, we have something more akin to structured exception
handling in C++; we’re going to "call" something, and it may jump control flow
to an arbitrary location.  Well, not arbitrary.  Arbitrary would be an indirect
call through a pointer that could’ve been constructed from any number and may
or may not be a valid instruction (or meant to be interpreted as one, ie. a
"gadget.")  `asm goto` is like virtual method calls or structured expection
handling in C++ in that they all can only transfer control flow to a short list
of possible destinations.

You might be wondering what you can build with this, and why does the Linux
kernel care?  Turns out the Linux kernel has multiple forms of self modifying
code that it uses in multiple different scenarios.  If you do something like:

```c
asm goto(
  ".pushsection foo\n"
  ".long %l0\n"
  ".popsection\n"
:::comefrom);
comefrom:;
```

You can squirrel away the address of `comefrom` in an arbitrary non-standard
ELF section.  Then at runtime if you know how to find ELF sections, you can
lookup `foo` and find the address of `comefrom` and then either jump to it, or
modify the instructions it points to.  I’ve used this trick to turn indirect
calls into direct calls (which is super dangerous and has many gotchas).

Luckily, the Linux kernel itself is an ELF executable, with all the machinery
for finding sections (it needs to perform relocations on itself at runtime,
after all), though it does something even simpler with the help of some linker
script magic as we’ll see.

[This LWN article](https://lwn.net/Articles/412072/) sums up the Linux’s
kernel’s original use case perfectly.

The kernel uses this for replacing runtime evaluation of conditionals with
either unconditional jumps or nop sleds when tracing, which are relatively
"expensive" to change when enabling or disabling tracing (requires machine wide
synchronization), but has minimally low overhead otherwise at runtime; just
enough nops in a sled to fit a small unconditional relative jump instruction
otherwise.  We can further tell the compiler whether the condition was likely
taken or not, which further influences codegen.

For patching in and out unconditional jumps with nop sleds, the kernel stores
an array of `struct jump_entry` in a custom  ELF section `.jump_table`, which
are triplets of:

1. The address of the start of the conditional or nop sled (you can initialize
   the branch to be in or out).  The `code` member of `struct jump_entry`.
2. The address of the label to potentially jump to. The `target` member of
   `struct jump_entry`.  For the case of whether a conditional evaluates to
   true or false.
3. A combination of the address of a global value representing the condition,
   and whether the branch is likely to be taken or not.  This is the `key`
   member of `struct jump_entry`.

The kernel uses pointer compression for 1 and 2 above, for architectures that
define `CONFIG_HAVE_ARCH_JUMP_LABEL_RELATIVE` as documented near the end of
[Documentation/x86/exception-tables.rst](https://www.kernel.org/doc/html/latest/x86/exception-tables.html).

The kernel uses pointer packing for 3 above, to pack whether the branch is
likely taken or not and the address of a `struct static_key` as documented in
an ascii art table near the end of include/linux/jump_label.h.  The pointed-to
`struct static_key` then uses pointer packing again to discriminate members of
an anonymous union, as documented in a comment within the definition of `struct
static_key` in include/linux/jump_label.h.

Naturally, helper functions exist and must be used for the above 3 cases to
reconstitute pointers from these values.

A custom linker script then defines two symbols that mark the beginning and end
of the section.  These symbols are forward declared in C as symbols with
`extern` linkage, then used to set boundaries when iterating the array of
`struct jump_entry` instances, when initializing the keys and when finding an
entry to patch.

Let’s take a quick peek at one architecture’s implementation of creating the
array of `struct jump_entry` in `.jump_table`, here’s
[arm64's implementation](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/arm64/kernel/jump_label.c):

```c
static inline __attribute__((always_inline)) bool arch_static_branch(struct static_key *key,
                                                                     bool branch)
{
	asm volatile goto (
		"1:	nop					\n\t"
		 "	.pushsection	__jump_table, \"aw\"	\n\t"
		 "	.align		3			\n\t"
		 "	.long		1b - ., %l[l_yes] - .	\n\t"
		 "	.quad		%c0 - .			\n\t"
		 "	.popsection				\n\t"
		 :  :  "i"(&((char *)key)[branch]) :  : l_yes);

	return false;
l_yes:
	return true;
}
```

There’s a lot going on here, so let’s take a look.  `1:` is a local label for
references within the asm block; it will get a temporary symbol name when
emitted.  `1:` points to a literal nop sled, but after the nop sled is the C
code following the `asm goto` statement.  That’s because the inline asm uses
the `.pushsection` directive to store the following data in an ELF section
that’s not `.text`.  We set the alignment of elements, then store two 32b
values and one 64b.  The `.long` directive has a comma that’s easy to miss, so
there’s two, and they’re compressed (`- .`) or made relative offsets of the
current location.  The first is the address of the beginning of the nop sled.
`1b` means local label named `1` searching `b`ackwards.  Finally, we store a
pointer to the `struct static_key` using pointer packing to add whether we’re
likely to take the branch or not.  The accessor functions will reconstruct the
two separate values correctly.

All this documentation is scattered throughout:

- [Documentation/static-keys.txt](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/static-keys.txt)
- [Documentation/x86/exception-tables.rst](https://www.kernel.org/doc/html/latest/x86/exception-tables.html)
- [include/linux/jump_label.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/jump_label.h)
- [kernel/jump_label.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/jump_label.c)
- arch/{$ARCH}/include/asm/jump_label.h
- arch/{$ARCH}/kernel/jump_label.c

In fact, once you know this trick of using `.pushsection` in extended inline
assembly and storing addresses of data, then using linker defined symbols to
delineate section boundaries for quick searching and iteration, we start to see
this pattern occur all throughout the kernel (with or without `asm goto`).
[This LWN article](https://lwn.net/Articles/531148/) discusses the trick and
the many custom ELF sections of a Linux image well.

Exception tables in the kernel in fact use very similar tricks of storing
addresses in custom ELF sections, `__ex_tables` and `.fixups` via inline
assembly.  The Linux kernel also sorts this data in the `__ex_table` section at
boot or even possibly post-link of the kernel image via BUILDTIME_TABLE_SORT,
so that at runtime the lookup of the exception handler can be done in log(N)
time via binary search! The .fixup also captures the address of the instruction
after the one that caused the exception, in order to possibly return control
flow to after successfully handling the exception.

"Alternatives" use this for patching in instructions that take advantage of ISA
extensions if we detect support for them at runtime.

A lot of kernel interfaces use function pointers that are written to once, then
either rarely or never modified.  It would be nice to replace these indirect
calls with direct calls.  In fact,
[patches have been proposed](https://lwn.net/ml/linux-kernel/20200324135603.483964896@infradead.org/)
to lower the overhead of the Spectre & Meltdown mitigations by doing just that.


Anyways, back to our story of debugging...

From here, I changed course and pursued another lead.  I had recently taught
LLVM’s inliner how to inline `asm goto` (or more so, when it was considered
safe to do so).  It seemed that LLVM’s inliner was not always respecting
`__attribute__((always_inline))` and could simply decide it wasn’t going to
perform an inline substitution.  (The inliner is a complex system; a large
analysis of multiple inputs distilled into a single yes/no signal, and all the
machinery necessary to perform such a code transformation).  The C standard (§
6.7.4
[ISO/IEC 9899:202x](http://www.open-std.org/jtc1/sc22/wg14/www/docs/n2479.pdf))
says compilers are allowed to make their own decisions in regards to inline
substitution, so it’s generally more conservative to just say "no" when
presented with a highly complex or unusual case.

When the "always inline" function wasn’t inlined, it was no longer semantically
valid, since it was passing its parameters as input to the inline asm using the
"i" machine agnostic constraint for integral literals, amongst other
questionable uses of `__attribute__((always_inline))` within the kernel.

I was working around this (before I fixed LLVM) by changing the
`__attribute__((always_inline)` functions into macros (because the preprocessor
doesn’t have the ability to silently fail to transform as the inliner does).
But everything was working when I did that; the kernel booted just fine.  Had I
regressed something when inlining?  Was there a corner case I wasn’t thinking
of, which happens all the time in compiler development?  Was the compiler
haunted?  Was my code bad? Probably. (Porque no los dos?)

I start bisecting object files used to link the kernel image, mixing code that
is either called a static always inline vs a macro, and I narrow it down to 4
object files.

- arch/x86/kernel/tsc.o
- kernel/time/hrtimer.o
- kernel/time/timer.o
- kernel/sched/clock.o

Reading the time stamp counter! No wonder the kernel is failing so early;
initializing the clocks is one of the earlier tasks the kernel cares about.  A
preemptive multitasking operating system is obsessed with keeping track of
time; you spend up your time slice and you’re scheduled out.

But why would a `static inline __attribute__((always_inline))` function fail,
but succeed when the function was converted to a macro?

I mentioned this to my colleague Bill Wendling, who spotted a subtle
distinction in LLVM’s IR between the `static inline
__attribute__((always_inline))` version of the functions (and their call sites)
vs the macro.  Via
[an email to the list](https://lists.llvm.org/pipermail/llvm-dev/2019-April/131518.html):

> The code below is triggering some weird behavior that's different from how
> gcc treats this inline asm. Clang keeps the original type of "loc" as "bool",
> which generates an "i1 true" after inlining. So far so good.  However, during
> ISEL, the "true" is converted to a signed integer. So when it's evaluated,
> the result is this:
>
>
>     .quad (42+(-1))-.Ltmp0
>
>
> (notice the "-1"). GCC emits a positive one instead:
>
>
>     .quad 42 + 1 - .Ltmp0
>
>
> I'm not sure where the problem lies. Should the inline asm promote the "i1"
> to "i32" during ISEL? Should it be promoted during inlining? Is there a
> situation where we require the value to be "i1"?
>
>
> -bw

```c
typedef _Bool bool;


static inline
__attribute__((__always_inline__))
bool bar(bool loc) {
        asm(".quad 42 + %c0 - .\n\t" : : "i" (loc));
        return 1;
}


int foo(void) {
        return bar(1);
}
```
Krzysztof Parzyszek [responded the next day](https://lists.llvm.org/pipermail/llvm-dev/2019-April/131526.html).

> This is a bug in X86's ISel lowering: it does not take "getBooleanContents"
> into account when extending the immediate value to 64 bits."

Oh, shit!  LLVM’s IR has support for arbitrary width integers which is fine for
a high level language.  Because real machines typically don’t have support for
such integers of arbitrary width, the compiler typically has to find legal
widths for these integers (we say it "legalizes the types") during lowering
from the high level abstract IR to low level concrete machine code.

To legalize a one bit integer into a 64 bit integer, we have to either zero
extend or sign extend it.  Generally, if we know the signedness of a number, we
sign extend signed integers to preserve the signedness of the uppermost bit, or
zero extend unsigned integers which don’t have a signedness bit to preserve.

But what happens when you have a boolean represented as a signed 1 bit number,
and you choose to sign extend it? `0x00` becomes `0x0000000000000000` which is
fine, but `0x01` becomes `0xFFFFFFFFFFFFFFFF`, ie. `-1`.  So when you expected
1, but instead got a -1, then you’re off by 2.  Preceding to use that in an
address calculation is going to result in some spooky bugs.

Recalling our inline `asm goto`, we we’re using this boolean to construct an
instance of a `struct jump_entry`’s `key` member, which was using pointer
packing to both refer to a global address and store whether the branch was
likely taken or not in the LSB.  When the value of `branch` was 0, we were
fine. But when `branch` was 1 and we sign extended it to -1, we kept the LSB as
1 but messed up the address of a global variable, resulting in the helper
function unpacking the pointer to a global `struct static_key` producing a bad
pointer.  Since the bottom two bits were dropped reconstituting the pointer, a
hypothetical value of 0x1001 would become 0xFFC (0x1001 - 2 & ~3) which would
be wrong by 5 bytes. Thus we were interpreting garbage as a pointer, which led
to cascaded failure.

In this case, it looks like Bill spotted that during instruction selection
something unexpected was occuring, and Krystof narrowed it down from there.
[Krystof had a fix available for x86](https://reviews.llvm.org/D60208), which
[Kees Cook later extended to all architectures](https://reviews.llvm.org/D60224).
Since then,
[Bill even extended LLVM’s implementation to allow for the mixed use of output constraints with `asm goto`](https://reviews.llvm.org/D69876),
something GCC doesn’t yet allow for, which is curious as Clang is now pushing a
GNU C extension further than GCC does.

(The true heroes of this story BTW are Alexander Ivchenko and Mikhail
Dvoretckii for
[providing the initial implementation](https://lists.llvm.org/pipermail/llvm-dev/2018-October/127239.html)
of `asm goto` support in LLVM, and
[Craig Topper](https://reviews.llvm.org/D53765) and
[Jennifer Yu](https://reviews.llvm.org/D56571) (all Intel) for carrying the
implementation across the finish line.  Kudos to Chandler Carruth for *noting
the irony and uncanny coincidence* that it was both Intel that
[regressed the x86 kernel build with Clang for over a year by requiring `asm goto` / CONFIG_JUMP_LABEL](https://lore.kernel.org/lkml/20180402095033.nfzcrmxvpm46dhbl@gmail.com/),
and provided an implementation for it in Clang.)

0 based array indexing is the source of a common programmer error; off by one.
In this case, sign extending a boolean led to our *off by two*. (Or were we off
by one at being off by one?)

+++

I’m lucky to have virtual machines and debuggers, and the ability to introspect
my compiler, but I’m not sure if all of those were available back when Linux
was first written.  For fun, I asked Linus Torvalds what early debugging of the
Linux kernel was like (reprinted with permission):

Nick:
> What do you do for testing?  Quick boot tests in QEMU are my smoke tests, but
> I'm always interested in leveling up my workflow.

Linus:
> I basically never do virtual machines. It happens - but mainly when chasing
> kvm bugs. With half of the kernel being drivers, I find the whole "run it in
> emulation" to be kind of pointless from an actual testing perspective.
>
> Yeah, qemu is useful for quick smoke-tests, and for all the automated stuff
> that gets run.
>
> But the automation happens on the big farms, and I don't do the quick smoke
> testing - if I get a pull requests from others, it had better be in good
> enough shape that something like that is pointless, and when I do my own
> development I prefer to think about the code and look at generated assembly
> over trying to debug a mistake.
>
> So if something doesn't work for me, that to me is a big red flag - I go and
> really stare at the code and try to understand it even better.  I am not a
> huge believer in debuggers, it's not how I've ever coded.
>
> I feel you get into a mindset where your code is determined by testing and
> "it works", rather than by actually thinking about it and knowing it and
> believing it is correct.
>
> But I probably just make excuses for "this is how I started, because
> emulation or debuggers just weren't an option originally, and now it's how I
> work".

Nick:
> One thing I am curious about is how the hell you ever debugged anything when
> you were starting out developing Linux?  Was the first step get something
> that could write out to the serial port?  (Do folks use serial debuggers on
> x86? USB? We use them often on aarch64.  Not for attaching a debugger, more
> so just for dmesg/printk).  Surely, it was some mix of "just think really
> hard about the code" then at some point you had something a little nicer?
> Graphics developers frequently have to contend with black screens and use
> various colors like all-red/all-green/all-blue when debugging as a lone
> signal of what's going wrong, which sucks, but is kind of funny.

Linus:
> Hey, when you make a mistake early on in protected mode, the end result is
> generally a triple fault - which results in an instant reboot.
>
> So my early debugging - before I had console output and printk - was
> literally "let's put an endless loop here", and if the machine locked up you
> were successful, and if it rebooted you knew you hadn't reached that point
> because something went wrong earlier.
>
> But it's not like doing VGA output was all _that_ complicated, so "write one
> character to the upper corner of the screen" came along pretty quickly. That
> gives you a positive "yeah, I _definitely_ got this far" marker, and not just
> a "hmm, maybe it locked up even before I got to my endless loop".
>
> Fun days.
