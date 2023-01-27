---
title: "Critical Edge Splitting"
date: 2023-01-27T09:16:05-08:00
comments: true
categories:
- compilers
- llvm
---
*A maximal length sequence of branch-free code that terminates with a branch or
jump is referred to as a **basic block**.*

*A basic block that branches to another forms an **edge** in the Control Flow
Graph (CFG).*

*The initial basic block starting an edge is the **predecessor**; it precedes
and is succeeded by the **successor** basic block.*

*An edge between basic blocks is considered a **critical edge** if the
predecessor has multiple successors, and the successor has multiple
predecessors.*

Given the below control flow graph, can you spot any critical edges?

![control flow graph](/images/cfg/cfg.svg)
---
![critical edge](/images/cfg/cfg2.svg)

Given the above control flow graph, we have one critical edge between BB0 and
BB3. (BB0 has multiple successors, but BB2 only has only one predecessor so the
edge between them is not critical. BB3 has multiple predecessors, but BB1 has
only one successor so the edge between them is also not critical).

Say we need to insert new code to execute along the critical edge. Do we put it
at the end of BB0 or the beginning of BB3?

Because BB0 has multiple successors, it might not be correct for such code to
execute at the end of BB0 when the branch to BB2 is taken. Because BB3 has
multiple predecessors, it would also not be correct to insert the new code at
the beginning of BB3 since it would execute should the incoming branch from BB1
be taken.

A common solution is to split the critical edge.
![critical edge splitting](/images/cfg/critical_edge_splitting.svg)

So when would we need to insert code along such an edge?

Let's say you'd like to add instrumentation to existing code to track whether a
branch is taken or not (to ascertain whether execution paths are covered by
testing). That's a boolean decision; taken or not. Sometimes a count is more
helpful for the purposes of profiling, which let's you calculate how hot or
cold an edge is relative to another. Such information might influence the
relative ordering of basic blocks when linearizing the control flow graph or
making decisions about whether to inline or even outline the contents of those
basic blocks.  Think about the control flow graph above. Where would you insert
the counter update if you hadn't split the critical edge?

Another important case of critical edge splitting involves translation out of
Static Single Assignment (SSA) form.  Cooper and Torczon's *Engineering a
Compiler* puts it pretty succinctly: *because modern processors do not
implement phi functions, the compiler needs to translate SSA form back into
executable code*.

Phi (Φ) functions are a common occurrence in SSA form Intermediate
Representations (IR). Here's a quick example in LLVM IR:

```llvm
define i32 @foo (i1 %x) {
entry:
  br i1 %x, label %bar, label %baz
bar:
  br label %quux
baz:
  br label %quux
quux:
  %ret = phi i32 [0, %bar], [1, %baz]
  ret i32 %ret
}
```
`@foo` is a simple identity function (never mind that the return type is wider
than the input, example chosen for brevity). But let's look at that `phi`
instruction. Check out LLVM's
[Language Reference](https://llvm.org/docs/LangRef.html#phi-instruction)
(aka "LangRef") for precise semantics of phi instructions, but in our example
we can see the type, followed by one-to-many tuples of value for a given
predecessor. So if we enter basic block `%quux` from `%bar`, the value of
`%ret` is `0`. Conversely, if we enter `%quux` from `%baz` then `%ret` gets the
value `1`.

As mentioned earlier, a phi is kind of a virtual operation. If there's only two
branches, we might be able to lower that to some kind of conditional move or
select operation (or a nest of such operations for multiple inputs). That can
quickly get out of hand though as the number of incoming edges to a phi grows,
particularly when the target Instruction Set Architecture (ISA) doesn't have
cmov/csel.

One of the seminal papers on SSA form proposed:

{{< blockquote author="Cytron, et al." link="https://www.cs.utexas.edu/~pingali/CS380C/2010/papers/ssaCytron.pdf" title="Efficiently Computing Static Single Assignment Form and the Control Dependence Graph" >}}
Naively, a k-input @-function at entrance to a node X can be replaced by k
ordinary assignments, one at the end of each control flow predecessor of X.
{{< /blockquote >}}

That might look like the following unoptimized aarch64 assembly (remember,
we're leaving SSA form for concrete machine instructions):

```asm
foo:
  tbz w0, #0, .Lbaz
  b   .Lbar
.Lbar:
  mov w0, #1
  b   .Lquux
.Lbaz:
  mov w0, #0
  b   .Lquux
.Lquux
  ret
```
So no phi, and we've inserted `mov` assignments into the predecessors of the
block that previously contained the phi.

Later research would later point to problems with this approach. The simple IR
example above doesn't contain a critical edge, but if it did, *where would you
place the copies* becomes a problem which can be solved via critical edge
splitting.

A modification of the above example demonstrates the problem:
```llvm
define i32 @foo (i1 %x) {
entry:
  br i1 %x, label %bar, label %baz

bar:
  br label %baz

baz:
  %ret = phi i32 [ 0, %entry ], [ 1, %bar ]
  ret i32 %ret
}
```
After naively putting copies in the predecessors *before* terminating
instructions (If you're curious *why can't we put them after*, consider whether
they would be executed as you'd expect):
```asm
foo:
  tbz w0, #0, .Lbaz
  mov w0, #0        // oops!
  b   .Lbar
.Lbar:
  mov w0, #1
  b   .Lbaz
.Lbaz
  ret
```
Where the edge from `%entry` to `%baz` is critical. If we split that first, we
get correct results.
```asm
foo:
  tbz w0, #0, .Lbaz_crit_edge
  b   w0, .Lbar
.Lbar:
  mov w0, #1
  b  .Lbaz
.Lbaz_crit_edge
  mov w0, #0
  b .Lbaz
.Lbaz
  ret
```

Critical edge splitting alone still has issues, but is still used in final
solutions for phi elimination.  You can search for more information on "the
lost-copy problem," "the swap problem," "Brigg's method," and "Sridhar's
method" for more information. I found Maarten Faddegon's 2011 Master's Thesis
*[SSA Back-Translation: Faster Results with Edge Splitting and Post Optimization](https://repository.tudelft.nl/islandora/object/uuid:a3bea0c0-fc6d-47a1-9b94-bf39df4aff8f?collection=education)*
to be an excellent and accessible read on the matter.

Some IR's like
[Swift's Intermediate Language](https://apple-swift.readthedocs.io/en/latest/SIL.html#basic-blocks)
(SIL) and
[Cranelift](https://github.com/bytecodealliance/wasmtime/blob/main/cranelift/docs/ir.md#static-single-assignment-form)
avoid complications with phi nodes entirely by using basic blocks that accept
arguments. Contrasting the two (phis vs basic block parameters) is something
I'd like to understand better.

Cooper and Torczon suggest that maybe you wouldn't want to split critical edges
when you have a backward edge, for example a hot loop, since you'd be inserting
branches that previously didn't exist. Though, Faddegon's paper states:

{{< blockquote author="Maarten Faddegon" link="https://repository.tudelft.nl/islandora/object/uuid:a3bea0c0-fc6d-47a1-9b94-bf39df4aff8f?collection=education" title="4.2 Previous Work Mentioning Phi Blocks">}}
As far as known by the author, there is no work about the effects of phiblocks
on the performance of the end result produced by the compiler.
{{< /blockquote >}}
(By phiblocks they are referring to the basic block that is inserted or
synthesized by critical edge splitting.) The author then demonstrates how critical edge splitting can improve performance of the prior two best approaches to SSA back translation by 3-5%.

For further real world uses of critical edge splitting, I recommend taking a
look at callers of `llvm::SplitCriticalEdge`, `llvm::SplitKnownCriticalEdge`,
and `MachineBasicBlock::SplitCriticalEdge`.  General edge splitting makes use
of `llvm::SplitKnownCriticalEdge`, so the number of references quickly grows
too large to enumerate reasonably here.  That critical edge splitting is a
fallible operation in LLVM is perhaps a story for another day.

[A recent feature I've been working on in LLVM](https://discourse.llvm.org/t/rfc-syncing-asm-goto-with-outputs-with-gcc/65453/6?u=nickdesaulniers)
also requires critical edge splitting. Had I done
any research before starting on the project, I might have found that Faddegon's
paper references "Boissinot's *Exotic Terminator Problem*."

Turns out, in 2008 Boissinot, et al. published a paper,
[Revisiting Out-of-SSA Translation for Correctness, Code Quality, and Efficiency](https://hal.inria.fr/inria-00349925v1/document),
detailing control flow instructions which also modify variables. Modeling this
in SSA is not easy. One approach suggested was critical edge splitting (you can
read the paper which I will link to below for the other approach).

That's pretty much
[the same problem I was trying to solve](https://discourse.llvm.org/t/rfc-syncing-asm-goto-with-outputs-with-gcc/65453/6?u=nickdesaulniers),
and ultimately
[the same solution I pursued](https://reviews.llvm.org/D139872).
I just wish I had had all of the above information researched BEFORE having had
started pursuing multiple different solutions which resulted in many dead ends.
I found previous search results related to *critical edge splitting* to be
unsatisfactory. Hopefully this post was more informative and relatively
concise.

If you're curious to learn more, here are the references I recommend:
- *[Global Value Numbers and Redundant Computations](https://classes.engineering.wustl.edu/~cytron/cs531/Resources/Papers/valnum.pdf)* - Rosen, et al. - 1988
- *[Efficiently Computing Static Single Assignment Form and the Control Dependence Graph](https://www.cs.utexas.edu/~pingali/CS380C/2010/papers/ssaCytron.pdf)* - Cytron, et al. - 1991
- *[Partial Redundancy Elimination](https://people.cs.umass.edu/~emery/classes/cmpsci710-spring2004/lecture12-pre.pdf)* - Berger - 2003
- *[Revisiting Out-of-SSA Translation for Correctness, Code Quality, and Efficiency](https://hal.inria.fr/inria-00349925v1/document)* - Boissinot, et al. - 2008
- *[Revisiting Out-of-SSA Translation for Correctness, Code Quality, and Efficiency For both Static and JIT Compilation](https://compilers.cs.uni-saarland.de/ssasem/talks/Alain.Darte.pdf)* - Boissinot, et al. - 2009
- *[Comparison and Evaluation of Back Translation Algorithms for Static Single Assignment Form](https://citeseerx.ist.psu.edu/document?repid=rep1&type=pdf&doi=88c847191bb6241f66eef02ae0d54a00cfbbdb61)* - Sassa, et al. - 2009
- *[SSA Back-Translation: Faster Results with Edge Splitting and Post Optimization](https://repository.tudelft.nl/islandora/object/uuid:a3bea0c0-fc6d-47a1-9b94-bf39df4aff8f/datastream/OBJ/download)* - Faddegon - 2011
- *Engineering a Compiler 2nd ed.* - Cooper & Torczon - 2012 (Note to self: buy 3rd ed. when they make a hardcover!)
- *[Conversion from SSA](https://www.inf.ed.ac.uk/teaching/courses/copt/lecture-4-from-ssa.pdf)* - Leather - 2019
- *[Partial Redundancy Elimination](https://www.cs.cmu.edu/afs/cs/academic/class/15411-f20/www/lec/17-pre.pdf)* - Goldstein - 2020
- *[Static Single Assignment Form](https://homepages.dcc.ufmg.br/~fernando/classes/dcc888/ementa/slides/StaticSingleAssignment.pptx)* - Pereira
- *[Introduction to PRE](https://piazza.com/class_profile/get_resource/hzkq9i9o1ec222/i1jophwv6ovlu)*
