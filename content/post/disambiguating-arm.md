---
title: "Disambiguating Arm, Arm ARM, Armv9, ARM9, ARM64, Aarch64, A64, A78, ..."
url: "blog/2023/03/10/disambiguating-arm"
date: 2023-03-10T11:39:11-08:00
categories:
- ARM
---
If you're new to the Arm ecosystem, consider this a quick primer on terms you
likely have seen before but might have questions about.

The **Arm** architecture is a family of Reduced Instruction Set Architectures
(RISC) with simple addressing modes. Data processing is done on register
operands otherwise relying on loads and stores to move data into and out of
registers.

**Arm** Limited,
[the British company](https://www.arm.com/company),
stewards the Arm architecture.

ARM is a legacy acronym for Acorn RISC Machine, then Advanced RISC Machines.
As we'll see, with new advancements in the architecture, previous terms for
things sometimes get renamed.

[The Arm Architectural Reference Manual for A-profile architecture](https://developer.arm.com/documentation/ddi0487/latest),
affectionately referred to as the **Arm ARM**, is *the* programming manual for
the architecture. If you're doing anything with Arm assembly, you probably have
this reference nearby.

**[Armv9](https://www.anandtech.com/show/16584/arm-announces-armv9-architecture)**
is the latest (as of this writing) in the family of architectures,
featuring additions such as newer scalable SIMD vector (SVE2) and matrix
(SME/SME2) operations and tracing functionality.

**Armv9.4-A** is the latest batch of extensions to Armv9. These extensions are
documented in the Arm ARM.  Some extensions are optional when introduced and
many become mandatory in future revisions if they weren't  already when
introduced.

The *A* in **Armv9-A** denotes the "Application Profile." These support virtual
memory via memory management units, and are what you're likely to find on any
Arm systems such as a phone, laptop, or server. There's also the "R" profile
for applications with real time system requirements, and "M" profiles which
you're more likely to find in microcontrollers which lack MMUs. The three
*[architectural profiles](https://developer.arm.com/documentation/dui0471/m/key-features-of-arm-architecture-versions/arm-architecture-profiles)*
are A, R, and M.

**AArch64** is an *execution state* and was one of the larger additions with
the introduction of ARMv8, which added support for 64b registers (31 general
purpose registers, dedicated 64b stack pointer, 64b program counter that cannot
be written to other than by branches or exceptions, and a zero-value
[pseudo-register](https://developer.arm.com/documentation/den0024/a/An-Introduction-to-the-ARMv8-Instruction-Sets/The-ARMv8-instruction-sets/Registers))
and addressing. At the same time, the **AArch32** execution state
was coined to refer to the legacy 32b functionality that folks were familiar
with from ARMv7 (15 32b GPRs, no dedicated SP, PC is writable).

Curiously, the Arm ARM doesn't mention the term **ARM64**; that seems to be a
term preferred by
[Apple](https://developer.apple.com/documentation/xcode/writing-arm64-code-for-apple-platforms),
[Microsoft](https://learn.microsoft.com/en-us/windows/arm/overview),
and
[Linus Torvalds](https://lore.kernel.org/lkml/CA+55aFxL6uEre-c=JrhPfts=7BGmhb2Js1c2ZGkTH8F=+rEWDg@mail.gmail.com/)
(that thread will always make me laugh, the maintainers of the port ultimately
decided to use
[*arm64* in the tree](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/arm64).
The name ultimately makes sense; the arm64
Linux kernel port can execute userspace code in AArch64 or AArch32 *execution
states*, though the kernel itself is AArch64-only).

If you want to learn about the calling convention (arguments are passed in
which registers) used on these Arm systems, you might read the *Procedure Call
Standard for the Arm Architecture* (aka **AAPCS**), which is published along
with other documentation related to the ABI
[here](https://github.com/ARM-software/abi-aa/releases).
This made the previous *APCS* and *TPCS* standards obsolete. Apple platforms
[diverge](https://developer.apple.com/documentation/xcode/writing-arm64-code-for-apple-platforms)
from Arm's ABI in specific ways. Microsoft also
[has docs](https://learn.microsoft.com/en-us/cpp/build/arm64-windows-abi-conventions)
(starting with a nice definition list like this post) on their ABI for Windows.

**A64** is the instruction set introduced with *AArch64*. In fact, it is the
only *instruction set* supported by *AArch64*. While registers in the *AArch64
execution state* are 64b, the instructions themselves are still only 32b (fixed
width).  **A32** now refers to the older ISA, which was also 32b fixed width
while T32 refers to the mixed 32b and 16b Thumb2 instructions. You may be
familiar with those ISAs if you've worked with ARMv7 or older devices. *A64* is
a clean break from *A32* and is a familiar but different ISA. For instance,
much fewer instructions support predication in *A64* than *A32*.

Not to be confused with *A64*, you might hear someone refer to a core as "being
an *A78*," or more formally **Cortex-A78**. Not only does Arm design the Arm
architecture, but
[they also design](https://www.anandtech.com/show/7112/the-arm-diaries-part-1-how-arms-business-model-works)
implementations of the architecture which we call micro architectures.
Regardless of the number that follows, if you see the terms *Cortex* or
*Neoverse*, those are Arm-designed microarchitectures of the Arm architecture.
*Cortex-A78* for instance implements up to ARMv8.3 extensions.  Wikipedia has
[a template](https://en.wikipedia.org/wiki/Template:Application_ARM-based_chips)
that is a quick reference to the most recent Arm microarchitectures.  Before we
can talk more about microarchitectures in the Arm realm, we need to detour to
topologies.

*DynamIQ* (and *big.LITTLE* before that) build upon the idea of using heterogeneous
(different) cores rather than homogeneous (similar) cores for multi-core
systems. I'm not sure this could still be considered *symmetric* multiprocessing.
The advantage of this design is the flexibility to be good at different things
at different times. We want large power hungry out of order processors to
improve performance when we need it, but we might prefer slower in-order cores
to help save power consumption (which would improve battery life). It's
interesting to see Intel doing something vaguely similar with the introduction
of *performance* and *efficiency* cores in their
[Alder Lake](https://fuse.wikichip.org/news/6115/intel-unveils-alder-lake-next-generation-mainstream-heterogeneous-multi-core-soc/)
microarchitecture.

By digging through the Technical Reference Manuals published by Arm for various
microarchitectures, we can see an interesting evolution of support for various
*execution states* with regards to various exception levels over time.

- [A55](https://developer.arm.com/documentation/100442/0200/Functional-description/Introduction/Features):
  "Both the AArch32 and AArch64 execution states at all Exception levels (EL0
  to EL3)."
- [X1](https://developer.arm.com/documentation/101433/r1p2/Functional-description/Introduction/Features):
  "AArch32 Execution state at Exception level EL0 only. AArch64 Execution
  state at all Exception levels (EL0 to EL3)"
- [X3](https://developer.arm.com/documentation/101593/0102/The-Cortex-X3--core/Cortex-X3--core-features):
  "AArch64 Execution state at all Exception levels, EL0 to EL3." [i.e. no
  AArch32 support]

If an SoC were to be composed of heterogeneous cores with varying levels of
AArch32 support, that
[would place interesting constraints on the operating system's process scheduler](https://blog.esper.io/android-dessert-bites-3-road-to-64-bit-3123759/);
you can't run an AArch32 program on a core that doesn't support it!

---

Below are some more legacy terms. They might still be relevant, depending on
how old some systems you still support are.

**ARM9** (not to be confused with *Armv9*, the version of the architecture) is
a family of cores, some implementing ARMv4t, some ARMv5.

*[StrongARM](https://en.wikipedia.org/wiki/StrongARM)*
was a series of ARMv4 CPUs built by Digital Equipment Corporation;
Intel acquired this IP as part of a settlement of a lawsuit, and eventually
designed their own ARMv5 microarchitecture called
*[XScale](https://en.wikipedia.org/wiki/XScale)*.
Eventually Intel
[sold](https://en.wikipedia.org/wiki/XScale#Sale_of_PXA_processor_line)
the PXA SoC family which was using XScale to Marvell.  One wonders
[what the world may have looked like](https://techcrunch.com/2016/05/17/how-intel-missed-the-iphone-revolution/)
had Intel stuck with XScale in addition to or instead of Atom.

ARMv4t introduced a compressed instruction set called **Thumb**. Instructions were
16b fixed width (that said, there were some oddities like
[BL and BLX that were actually encoded as a pair of 16b instructions each](https://developer.arm.com/documentation/ddi0308/d/Thumb-Instructions/Alphabetical-list-of-Thumb-instructions/BL--BLX--immediate-);
implementations had to take care that exception returns worked correctly if an
exception occurred in the middle of the pair; it was implementation defined if
that could even occur).

ARMv6t2 introduced **Thumb2** which added more instructions including some 32b
wide instructions to support wider immediates, new instruction suffixes to
differentiate between *narrow* vs *wide* encodings, and a Unified Assembly
Language (UAL) that made it easier to write assembler that was valid in Arm or
Thumb mode. This made Thumb no longer fixed width though. The introduction of
*execution states* with ARMv8 renamed Thumb to *T32*; there was no such *T32*
term when these instructions were introduced!

You may come across the term *aarch64be* being used in the context of toolchains,
which is referring to big-endian.  Arm has supported bi-endianness
[since ARMv4](https://doc.rust-lang.org/rustc/platform-support/armeb-unknown-linux-gnueabi.html),
though most platforms these days use Arm in little-endian endian configuration.
Big-endian is more common in networking appliances since network byte order is
BE. `-mlittle-endian` and `-mbig-endian` are the compiler flags one might use
to control codegen.  ARMv4 and v5 supported a **BE-32** *bus byte ordering*.
Code linked with
[`--be32`](https://developer.arm.com/documentation/dui0493/g/linker-command-line-options/--be32)
produced big-endian code and data. ARMv6 added a new
bus byte ordering called *BE-8*.
[`--be8`](https://developer.arm.com/documentation/dui0493/g/linker-command-line-options/--be8)
produced little-endian code and big-endian data (the compiler would emit
big-endian code for relocatable files when built with `-big-endian`, then the
linker would convert these to little endian when `--be8` was used. This allowed
compilers to not worry about byte-reversing code regardless of what bus byte
ordering was to be used at the expense of linker complexity). ARMv6 had both
[BE-32 and BE-8 bus byte orderings](https://developer.arm.com/documentation/ddi0290/g/unaligned-and-mixed-endian-data-access-support/mixed-endian-access-support/differences-between-be-32-and-be-8-buses)
(the older BE-32 became optional), though
[ARMv7 removed support for BE-32](https://developer.arm.com/documentation/ddi0406/cb/Appendixes/Deprecated-and-Obsolete-Features/Obsolete-features/Support-for-BE-32-endianness-model).
[This post](https://blog.richliu.com/2010/04/08/907/arm11-be8-and-be32/)
shows why BE-8 replaced BE-32; it was simpler to support systems of both
endiannesses if we used little endian instructions and had the memory bus
reorder the bytes on access.  ELF uses the file format identifiers
elf64-littleaarch64, elf64-bigaarch64, elf32-littlearm, and elf32-bigarm;
though those identifiers don't appear in
[ELF for the Arm {64-bit} Architecture](https://github.com/ARM-software/abi-aa/releases).

That's a quick glossary over common terms related to the Arm ecosystem.
Hopefully in a follow up post we can review terms like VFP, Neon, OABI, and
EABI, but these are enough for now.

Many thanks to my friends Peter Smith, Kristof Beyls, and Mark Brown of Arm,
Arnd Bergmann of Linaro, and Ard Biesheuvel of Google, for proofreading drafts
of this post and supplying insightful feedback. Coincidentally, while I was
taking my time editing this post, my friend and colleague Fangrui Song
[beat me to the punch](https://maskray.me/blog/2023-03-05-linker-notes-on-aarch64)
which another great blog post touching on very similar topics; you should check
out
[his blog](https://maskray.me/blog/)
if you like this kind of content!
