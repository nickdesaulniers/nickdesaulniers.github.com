+++
title = "F() vs F(void) in C vs C++"
date = "2019-05-12"
slug = "2019/05/12/f-vs-f-void-in-c-vs-c-plus-plus"
Categories = ["C", "C++", "syntax"]
+++
TL;DR

Prefer `f(void)` in C to *potentially* save a 2B instruction per function call
when targeting x86_64 as a micro-optimization. `-Wstrict-prototypes` can help.
Doesn’t matter for C++.

## The Problem

While messing around with some C code in
[godbolt Compiler Explorer](http://godbolt.org),
I kept noticing a particular funny case.  It seemed with my small test cases
that sometimes function calls would zero out the return register before calling
a function that took no arguments, but other times not.  Upon closer
inspection, it seemed like a difference between function definitions,
particularly `f()` vs `f(void)`.  For example, the following C code:

```c
int foo();
int bar(void);

int baz() {
  foo();
  bar();
  return 0;
}
```
would generate the following assembly:
```gas
baz:
  pushq %rax # realign stack for callq
  xorl %eax, %eax # zero %al, non variadic
  callq foo
  callq bar # why you no zero %al?
  xorl %eax, %eax
  popq %rcx
  retq
```
In particular, focus on the call the `foo` vs the call to `bar`.  `foo` is
preceded with `xorl %eax, %eax` (X ^ X == 0, and is the shortest encoding for
an instruction that zeroes a register on the variable length encoded x86_64,
which is why its used a lot such as in setting the return value).  (If you’re
curious about the pushq/popq, see
[point #1](/blog/2014/04/18/lets-write-some-x86-64/).)
Now I’ve seen zeroing before (see
[point #3](/blog/2014/04/18/lets-write-some-x86-64/)
and remember that `%al` is the lowest byte of `%eax` and `%rax`), but if it was
done for the call to `foo`, then why was it not additionally done for the call
to `bar`? `%eax` being x86_64’s return register for the C ABI should be treated
as call clobbered.  So if you set it, then made a function call that may have
clobbered it (and you can’t deduce otherwise), then wouldn’t you have to reset
it to make an additional function call?

Let’s look at a few more cases and see if we can find the pattern.  Let’s take
a look at 2 sequential calls to `foo` vs 2 sequential calls to `bar`:
```c
int foo();
int quux() {
  foo(); // notice %eax is always zeroed
  foo(); // notice %eax is always zeroed
  return 0;
}
```
```gas
quux:
  pushq %rax
  xorl %eax, %eax
  callq foo
  xorl %eax, %eax
  callq foo
  xorl %eax, %eax
  popq %rcx
  retq
```
```c
int bar(void);
int quuz() {
  bar(); // notice %eax is not zeroed
  bar(); // notice %eax is not zeroed
  return 0;
}
```
```gas
quuz:
  pushq %rax
  callq bar
  callq bar
  xorl %eax, %eax
  popq %rcx
  retq
```

So it should be pretty clear now that the pattern is `f(void)` does not
generate the `xorl %eax, %eax`, while `f()` does.  What gives, aren’t they
declaring `f` the same; a function that takes no parameters?  Unfortunately, in
C the answer is no, and C and C++ differ here.

## An explanation

`f()` is not necessarily "`f` takes no arguments" but more of "I’m not telling
you what arguments `f` takes (but it’s not variadic)."  Consider this perfectly
legal C and C++ code:
```c
int foo();
int foo(int x) { return 42; }
```
It seems that C++ inherited this from C, but only in C++ does `f()` seems to
have the semantics of "`f` takes no arguments," as the previous examples all no
longer have the `xorl %eax, %eax`.  Same for `f(void)` in C or C++. That's
because `foo()` and `foo(int)` are two different function in C++ thanks to
function overloading (thanks reddit user /u/OldWolf2). Also, it seems that C
supported this difference for backwards compatibility w/ K & R C.

```c
int bar(void);
int bar(int x) { return x + 42; }
```
Is an error in C, but in C++ thanks to function overloading these are two
separate functions! (`_Z3barv` vs `_Z3bari`). (Thanks HN user
[pdpi](https://news.ycombinator.com/item?id=19895079), for helping me
understand this. Cunningham's Law ftw.)

Needless to say, If you write code like that where your function declarations
and definitions do not match, you will be put in prison.
[Do not pass go, do not collect $200](https://youtu.be/D2ydY5sBnIg?t=97)).
[Control flow integrity](https://clang.llvm.org/docs/ControlFlowIntegrity.html)
analysis is particularly sensitive to these cases, manifesting in runtime
crashes.

## What could a sufficiently smart compiler do to help?

`-Wall` and `-Wextra` will just flag the `-Wunused-parameter`.  We need the
help of `-Wmissing-prototypes` to flag the mismatch between declaration and
definition. (An aside; I had a hard time remembering which was the declaration
and which was the definition when learning C++.  The mnemonic I came up with
and still use today is: think of definition as in muscle definition; where the
meat of the function is.  Declarations are just hot air.)  It’s not until we
get to `-Wstrict-prototypes` do we get a warning that we should use `f(void)`.
`-Wstrict-prototypes` is kind of a stylistic warning, so that’s why it’s not
part of `-Wall` or ` -Wextra`.  Stylistic warnings are in bikeshed territory
(\*cough\* `-Wparentheses` \*cough\*).

One issue with C and C++’s style of code sharing and encapsulation via headers
is that declarations often aren’t enough for the powerful analysis techniques
of production optimizing compilers (whether or not a pointer
["escapes"](https://jonasdevlieghere.com/escape-analysis-capture-tracking-in-llvm/)
is a big one that comes to mind).  Let’s see if a "sufficiently smart compiler"
could notice when we’ve declared `f()`, but via observation of the definition
of `f()` noticed that we really only needed the semantics of `f(void)`.

```c
int puts(const char*);
int __attribute__((noinline)) foo2() {
  puts("hello");
  return 0;
}

int quacks() {
  foo2();
  foo2();
  return 0;
}
```
```gas
quacks:
  pushq %rax
  callq foo2
  callq foo2
  xorl %eax, %eax
  popq %rcx
  retq
```

A ha! So by having the full definition of `foo2` in this case in the same
translation unit, Clang was able to deduce that `foo2` didn’t actually need the
semantics of `f()`, so it could skip the `xorl %eax, %eax` we’d seen for `f()`
style definitions earlier.  If we change `foo2` to a declaration (such as would
be the case if it was defined in an `external` translation unit, and its
declaration included via header), then Clang can no longer observe whether
`foo2` definition differs or not from the declaration.

So Clang can potentially save you a single instruction (`xorl %eax, %eax`)
whose encoding is only 2B, per function call to functions declared in the style
`f()`, but only IF the definition is in the same translation unit and doesn’t
differ from the declaration, and you happen to be targeting x86_64. \*deflated
whew\* But usually it can’t because it’s only been provided the declaration via
header.

## Conclusion

I certainly think `f()` is *prettier* than `f(void)` (so C++ got this right),
but pretty code may not always be the fastest and it’s not always
straightforward when to prefer one over the other.

So it seems that `f()` is ok for strictly C++ code.  For C or mixed C and C++,
`f(void)` might be better.

## References

- https://stackoverflow.com/q/6212665/1027966
- https://stackoverflow.com/q/693788/1027966
- https://stackoverflow.com/q/12643202/1027966
- https://stackoverflow.com/q/51032/1027966
- https://stackoverflow.com/q/416345/1027966
- https://stackoverflow.com/q/41803937/1027966
- http://david.tribble.com/text/cdiffs.htm#C99-empty-parm
- https://godbolt.org/z/nsgGpt
- https://godbolt.org/z/IORbBb
- https://godbolt.org/z/51s7K0
- https://bugs.llvm.org/show_bug.cgi?id=41851 (the maintainer of Clang cites
the relevant part of the C11 spec for this)

