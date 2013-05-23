---
layout: post
title: "Rust: Pattern Matching and the Option Type"
date: 2013-05-07 18:41
comments: true
categories: [Rust, Pattern Matching, Option Type, Null Pointer, C++]
published: true
---
The other day I was thinking about the function for performing dynamic memory
allocation in the C standard library, malloc. From the manual pages,
`If successful, the malloc() function returns a pointer to allocated memory.
If there is an error, it
returns a NULL pointer and sets errno to ENOMEM.` One of the most common errors
when using malloc is not checking for allocation failure.  The allocation is not
guaranteed to succeed and trying to use a NULL reference can lead to program
crashes.

So a common pattern we'll see is:

```c C
int* nums = (int*) malloc(sizeof(int));
if (nums == NULL) {
  // handle error
} else {
  *nums = 7;
  // operate on nums
  free(nums);
  nums = NULL;
}
```

Here we allocated space for an integer, cast the `void*` returned from malloc
to an `int*`, compared it against the `NULL` pointer, then freed the allocated
memory and removed the reference.

One of the big problems with the null pointer has to do with
safely dereferencing it.  In C, dereferencing the null pointer is undefined
and usually leads to a segfault and program crash.

It can be so unsafe to work with null pointers that
C. A. R. Hoare refers to them as his billion-dollar mistake:

{% blockquote C.A.R. Hoare, InfoQ http://www.infoq.com/presentations/Null-References-The-Billion-Dollar-Mistake-Tony-Hoare Null References: The Billion Dollar Mistake %}
I call it my billion-dollar mistake. It was the invention of the null reference in 1965. At that time, I was designing the first comprehensive type system for references in an object oriented language (ALGOL W). My goal was to ensure that all use of references should be absolutely safe, with checking performed automatically by the compiler. But I couldn't resist the temptation to put in a null reference, simply because it was so easy to implement. This has led to innumerable errors, vulnerabilities, and system crashes, which have probably caused a billion dollars of pain and damage in the last forty years.
{% endblockquote %}

So how do we represent the value of nothing?  With an integer, for example, your
first instinct might be to use 0 to refer to no value.  But 0 *is* a value, so
how do we represent the range of integer values but also whether there is a
value or not?

Rust, a systems programming language with a focus on safety and concurrency,
does not have the concept of a null pointer.  Instead, it has a different
construct to represent the absence of value, a whole other structure called an
[Option<T>](http://static.rust-lang.org/doc/core/option.html).  It is an
[enumerated type](http://static.rust-lang.org/doc/core/option.html#enum-option) that can
either be None (no value) or Some(T) (a specialization of type T).

So what the heck is an option type and how do we use it?  The option type is
a polymorphic type (generic type that can be specialized) that encapsulates
either an empty constructor or the constructor of the original data type.  Let's
take a trip down the rabbit hole to see how we use one.  First, let's look at
some C++ code, and then translate it to Rust.

```c++ option.cpp
#include <iostream> // cout, endl
#include <stdlib.h> // rand
#include <time.h> // time
using namespace std;

// If there is a 'random error' returns NULL
const int* const may_return_null () {
  srand(time(NULL));
  return rand() % 2 == 1 ? new int(666) : NULL;
}

int main () {
  // if else
  const int* const x = may_return_null();
  if (x) {
    // switch
    switch (*x) {
      case 777: cout << "Lucky Sevens"        << endl; break;
      case 666: cout << "Number of the Beast" << endl; break;
      case 42: cout  << "Meaning of Life"     << endl; break;
      default: cout  << "Nothing special"     << endl; break;
    }
  } else {
    cout << "No value" << endl;
  }
  
  // single if
  if (*x == 666) {
    cout << "Did I mention that Iron Maiden is my favorite band?" << endl;
  }
}

```

Let's step through this program line by line, starting in main.

```c++ Line 14
const int* const x = may_return_null();
```

Here we're calling a function that may return null, just like malloc!

```c++ Lines 8-9
srand(time(NULL));
return rand() % 2 == 1 ? new int(666) : NULL;
```

In the body of `may_return_null` we seed the random number generator, generate a random number, mod it by 2
(so it can either be 0 or 1, 50-50 chance, hopefully), then either return a
pointer pointing to memory allocated on the heap or the null pointer.  We also use the succinct
ternary operator, which gives us the power of a conditional statement in the
form of a concise expression.

```c++ Line 15
if (x) {
```

We check if the pointer is valid, that is that it is safe to use, as `NULL` is
falsy in a C++ conditional.  If it is, then we can safely dereference it.  Let's
switch on the (dereferenced) value.  Notice how we need to break to explicitly 
prevent fall through, though
[97% of the time that's what you intend](http://books.google.com/books?id=4vm2xK3yn34C&pg=PA37&lpg=PA38&dq=expert+c+fallthrough&source=bl&ots=Ho98ZhXF9X&sig=PebysT8-3zA_9B2aRkuvnz4mCmY&hl=en&sa=X&ei=j8CJUdeWMerJiwLQmICoDw&ved=0CC4Q6AEwAA#v=onepage&q=expert%20c%20fallthrough&f=false).

```c++ Lines 17-22
switch (*x) {
  case 777: cout << "Lucky Sevens"        << endl; break;
  case 666: cout << "Number of the Beast" << endl; break;
  case 42: cout  << "Meaning of Life"     << endl; break;
  default: cout  << "Nothing special"     << endl; break;
}
```

If the pointer was null, the else branch of the conditional would execute
printing a different result.

```c++ Line 24
cout << "No value" << endl;
```

Finally in we check the value pointed to.  Did I forget something here?  Save
that thought, we'll come back to it.

```c++ Lines 28-30
if (*x == 666) {
  cout << "Did I mention that Iron Maiden is my favorite band?" << endl;
}
```

Let's see my rough translation of this program into Rust.

```rust option.rs
use core::rand;

fn may_return_none () -> Option<int> {
  if rand::Rng().next() % 2 == 1 { Some(666) } else { None }
}

fn main () {
  let x: Option<int> = may_return_none();

  io::println(match x {
    Some(777) => "Lucky Sevens",
    Some(666) => "Number of the Beast",
    Some(422) => "Meaning of Life",
    Some(_) => "Nothing special",
    None => "No value"
  });

  match x {
    Some(666) => {
      io::println("Did I mention that Iron Maiden is my favorite Band?")
    },
    _ => {}
  };
}
```

Both programs, when compiled *should* randomly print either:

```
No Value
// or
Number of the Beast
Did I mention that Iron Maiden is my favorite Band?
```

Now let's walk through the Rust code, starting at main and compare it to the
equivalent C++ code.

```rust option.rs Line 8
let x: Option<int> = may_return_none();
```
```c++ option.cpp Line 14
const int* const x = may_return_null();
```

I read this Rust code as 'let x be type Option,
specialized to type int initialized
to the return value of may_return_none.'  I read the C++ as 'x is a const pointer
to a const integer initialized to the return value of may_return_none.'  It's
important to note that values and pointers in Rust default to being immutable,
where as in c++ they default to being mutable, which is why we need to be
explicit about their const'ness.  In Rust, we could declare x as being explicitly
mutable: `let mut x: ... = ...;`.  In both, we can also leave the explicit types
to be inferred by the compiler.

```rust option.rs Line 4
if rand::Rng().next() % 2 == 1 { Some(666) } else { None }
```
```c++ option.cpp Lines 8-9
srand(time(NULL));
return rand() % 2 == 1 ? new int(666) : NULL;
```

Now for the body of `may_return_none`.  Rust does not have a ternary operator,
so a single line `if {} else {}` block will have to do.  We also don't need
parentheses around the predicate in the conditional statement.  If we add them,
no harm is done, as we'd just be being redundant about order of operations.
There
are no semicolons in this expression, nor a return statement in this function.
Rust will return the last expression in a function or method if the semicolon is
left off.  Also there is no such thing as a conditional statement, only
conditional expressions; the if is itself something that can be evaluated and
assigned to a variable!  We see this later in the code where we pass the evaluation
of a match expression directly into a call of `io::println`.
Rust returns the evaluation of the if expression,
which is the evaluation of the branch's block specified by the predicate,
which will randomly be one of the two enumerated types of `Option<int>`, either
an encapsulation of a int whose literal value is `666`, `Some<666>`, or
the representation of no value, `None`.  I believe the RNG code has changed in 0.7,
this code was written in 0.6.

In Rust, to get the value out of an option type, we pattern match on it.  Pattern
matching is something I became familiar with through the
[Haskell](http://learnyouahaskell.com/syntax-in-functions#pattern-matching) programming language.  Pattern
matching is a powerful language construct that can entirely replace conditionals.
In fact, the one line if expression could have been written using a match
expression:

```rust
if rand::Rng().next() % 2 == 1 { Some(666) } else { None }
// or
match rand::Rng().next() % 2 { 1 => Some(666), _ => None }
``` 

The basic design pattern
for accessing the value of an option type in Rust looks like:

```rust
match x { // x: Option<T>
  Some(y) => { *y },
  None => {}
}
```

The curly braces are optional for one liners (no block needed).
Pattern matches have to be exhaustive.  That means I have to exhaust all
possibilities for what the deconstructed value could be.  You can use a branch
that looks like:

```rust
_ => 'everything else'
```

to catch everything else.  The underscore here means "every other possible case."
So by having to use a match expression (also not a
statement, as opposed to C++'s switch statement), which itself must be exhaustive,
Rust forces us to handle the case where the optional type is None!  This will
help us again in the future.
We also don't need parentheses around the predicate for the match expression, which
in this case is just a single variable, `x`.

In the value
`Some(_)`, the underscore has an additional meaning here that we would not use a
variable in the corresponding arm's block.  If we declared it as `Some(y)` we would get the warning:

```
option.rs:14:9: 14:11 warning: unused variable: `y`
option.rs:14     Some(y) => "Nothing special",
                      ^~
```

I hope back in my C++ code you spotted the fatal flaw.  On line 28, I just
dereferenced a raw pointer without checking its validity.  This is a violation
of memory safety.

```c++ option.cpp Line 28
if (*x == 666) {
```

When running the C++ code, instead of seeing `No value` printed to stdout in the
case of no value, a segfault occurs.

```
No value
[1]    80265 segmentation fault  ./option
```

What I should have done is something more like:

```c++ Line 28 corrected
if (x && *x == 666) {
```

But, the C++ compiler let me get away with not handling the case where the
pointer was invalid (even if doing nothing in the case of "handling" it).  By
leaving out the check for a valid pointer, I have instructed the machine to
behave the same or follow the same code path with and without a reference to a
valid memory location.  Let's see what happens when I don't handle the None case
by deleting line 15 or option.rs, `None => "No value"`:

```
option.rs:10:14: 16:3 error: non-exhaustive patterns: None not covered
option.rs:10   io::println(match x {
option.rs:11     Some(777) => "Lucky Sevens",
option.rs:12     Some(666) => "Number of the Beast",
option.rs:13     Some(42) => "Meaning of Life",
option.rs:14     Some(_) => "Nothing special"
```

Not only did the compiler prevent me from generating an executable, it told me
that a pattern was not exhaustive, explicitly which one, and what case that was
not covered.

Coming back to encoding a lack of value for an int, we had left off that 0 *is*
a valid value.  For instance, how should we represent any other integer divided
by 0, as an integer?
Signed integers use a single bit to store whether
their value is positive or negative, the tradeoff being the signed integers
represent up to one less power of two than unsigned integers.  Maybe we could use an
additional bit to represent valid or invalid, but this would again cost us in
terms of representable values.
The IEEE 754 floating point representation has encodings for [plus and minus
Infinity, plus and minus 0, and two kinds of NaN](https://en.wikipedia.org/wiki/IEEE_floating_point#Formats).
To solve the validity problem, we can use enumerated types, which in Rust occur
as specializations of the Option type.  It's up to the compiler to implement
either additional information for the type, or use raw null pointers.  And to
get the value back out of the Option type, we *must* handle the case where there
is no valid value.

The key takeaways that I wish to convey are:

1. In C and C++, I can use NULL to refer to a pointer that has no value.
1. A C/C++ compiler, such as clang, will allow me to compile code that violates
   memory safety, such as dereferencing NULL pointers.
2. Rust instead uses Option types, an enumeration of a specialized type or None.
3. Rust forces me to use a match statement to access the possible value of an
   Option type.
4. Pattern matching in Rust must be exhaustive.
5. The Rust compiler, rustc, forces me to handle the case where the pointer has
   no value, whereas the C++ compiler, clang++, did not.
6. All conditional and switch statements can be replaced with pattern matching.
