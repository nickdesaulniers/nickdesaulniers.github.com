---
layout: post
title: "Closures: JavaScript, Ruby, and Rust"
date: 2013-01-14 20:28
comments: true
published: false
categories: [Closure, JavaScript, Ruby, Rust]
---

It's all about closures.  Understanding scope is paramount to coding.
What can you access and what can't you access.  Closures allow us to
access variables that otherwise might be awkward to pass into a
function.  Closures can help us out of tricky situations, but can
confuse those from backgrounds with (typically) static languages that
may not support closing over variables.

Rust is an up and coming systems level programming language being
developed at Mozilla.  Let's take a look at the
sytax of closures in Rust, but first let's see how closures are
implemented in some other, more popular languages.

In JavaScript, closures are super common.  Here's a simple example.

{% gist 4536170 %}

JavaScript has function scope; the definition of a new function creates
a new scope.  But if a reference to an identifier is not found within
the local scope, the next outer scope is consulted and so on until it is
either found, or a ReferenceError is raised.  We could prevent this for
example by shadowing the variable name by naming it in the parameter
list:

{% gist 4536211 %}

Notice how in the above snippet, we cannot close over the outer x?  This
is called variable shadowing.  In JavaScript, no matter if you use a
function definition or function expression to define a function, as long
as you do not shadow a particular variable, then you may close over it.

Let's see how ruby handle's closures.

{% gist 4536257 %}

Oops.  Looks like Ruby doesn't support closures!  But I thought it was
supposed to be better than Java!  Well, it turns out Ruby *does* support
closures, it just has an alternate sytax for functions that close over
references to variables in the outer scope.  This alternate syntax
actually creates a new object containing the captured scope.  Lets take
advantage of Ruby's lambda's to close over variables:

{% gist 4536433 %}

Note, we could have also used the older lambda syntax, or even a Proc
object.  I won't cover the difference between the two here; there are
better blog posts on the differences.  My point is that ***not all
languages support closures with their default function declaration
syntax (like in JavaScript)***.  This provides a nice sytax into closures in Rust.

{% gist 4536519 %}

Note: Rust has some really nice type inference.  We can write the
previous snippet more succintly:

{% gist 4536527 %}

It might seem obvious that my point about different languages having
different syntaxes.  I guess a better stating of that point is that
***closure definitions differ from vanilla function definitions like
Ruby and as opposed to JavaScript***.

Rust aims to be a systems level programming language to replace C and
C++.  With closures, type inference, and a syntax that faintly reminds
me of Ruby's block lambda syntax, I'll take it!
