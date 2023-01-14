+++
title = "Closures: JavaScript, Ruby, and Rust"
date = "2013-01-14"
url = "blog/2013/01/14/closures-javascript"
Categories = ["closure", "javascript", "ruby", "rust"]
+++

It's all about [closures](http://en.wikipedia.org/wiki/Closure_%28computer_science%29).  Understanding [scope](http://en.wikipedia.org/wiki/Scope_%28computer_science%29) is paramount to coding.
What can you access and what can't you access.  Closures allow us to
access variables that otherwise might be awkward to pass into a
function.  Closures can help us out of tricky situations, but can
confuse those from backgrounds with (typically) [statically typed](http://en.wikipedia.org/wiki/Statically_typed#Static_typing) languages that
may not support closing over variables.

[Rust](http://www.rust-lang.org/) is an up and coming systems level programming language being
developed at [Mozilla](http://www.mozilla.org).  Let's take a look at the
syntax of closures in Rust, but first let's see how closures are
implemented in some other, more popular languages.

In [JavaScript](https://developer.mozilla.org/en-US/docs/JavaScript), closures are super common.  Here's a simple example.

```javascript
var x = 5;
 
function myClosure (y) {
  return x + 1;
};
 
console.log(myClosure(10)); // 6
```

JavaScript has function scope; the definition of a new function creates
a new scope.  But if a reference to an identifier is not found within
the local scope, the next outer scope is consulted and so on until it is
either found, or a ReferenceError is raised.  We could prevent this for
example by shadowing the variable name by naming it in the parameter
list:

```javascript
var x = 5;
 
function myClosure (x) {
  return x + 1;
};
 
console.log(myClosure(10)); // 11
```

Notice how in the above snippet, we cannot close over the outer x?  This
is called [variable
shadowing](http://en.wikipedia.org/wiki/Variable_shadowing).  In JavaScript, no matter if you use a
[function definition or function expression](http://stackoverflow.com/q/1013385/1027966) to define a function, as long
as you do not shadow a particular variable, then you may close over it.

Let's see how [Ruby](http://www.ruby-lang.org) handles closures.

```ruby
x = 5
 
def my_closure y
  x + 1
end
 
puts my_closure 10 # NameError: undefined local variable or method `x' for main:Object
```

Oops.  Looks like Ruby doesn't support closures!  But I thought it was
supposed to be easier to write than [Java](http://www.java.com)!  Well, it turns out Ruby *does* support
closures, it just has an alternate syntax for functions that close over
references to variables in the outer scope.  This alternate syntax
actually creates a new object containing the captured scope.  Lets take
advantage of Ruby's
[lambdas](http://en.wikipedia.org/wiki/Anonymous_function#Ruby) to close over variables:

```ruby
x = 5
 
my_closure = -> x do
  x + 1
end
 
puts my_closure.call 10 # 6
```

Note, we could have also used the older lambda syntax, or even a Proc
object.  I won't cover the difference between the two here; there are
[better blog posts on the differences](http://www.robertsosinski.com/2008/12/21/understanding-ruby-blocks-procs-and-lambdas/).  My point is that ***not all
languages support closures with their default function declaration
syntax (like in JavaScript)***.  This provides a nice syntax into closures in Rust.

```rust
fn main () {
  let x: int = 5;
 
  let my_closure = |_: int| -> int {
    x + 1
  };
 
  io::println(fmt!("%d",my_closure(10))); // 6
}
```

Note: Rust has some really nice [type inference](http://en.wikipedia.org/wiki/Type_inference).  We can write the
previous snippet more succinctly:

```rust
fn main () {
  //let x: int = 5;
  let x = 5;
 
  //let my_closure = |_: int| -> int {
  let my_closure = |_| {
    x + 1
  };
 
  io::println(fmt!("%?",my_closure(10))); // 6
}
```

It might seem obvious that my point about different languages having
different syntaxes.  I guess a better stating of that point is that
***closure definitions in Rust differ from vanilla function definitions like
Ruby and as opposed to JavaScript***.

Rust aims to be a systems level programming language to replace C and
C++.  With closures, type inference, and a syntax that faintly reminds
me of Ruby's block lambda syntax, I'll take it!
