+++
title = "Function Dot Prototype Dot Bind Edge Cases"
date = "2013-09-26"
slug = "2013/09/26/function-dot-prototype-dot-bind-edge-cases"
Categories = []
+++
ECMAScript 5's
[Function.prototype.bind](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/bind)
is a great tool that's implemented in all
[modern browser JavaScript engines](http://kangax.github.io/es5-compat-table/#Function.prototype.bind).
It allows you to modify the context,
[this](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/this),
of a function when it is evaluated in the future.  Knowing what `this` refers to
in various contexts is key to being a professional JavaScript developer; don't
show up to an interview without knowing all about it.

Here's a common use case that developers need to watch for.  Can you spot the
mistake?

```javascript
var person = "Bill";

var obj = {
  person: "Nick",
  hi: function () {
    console.log("Hi " + this.person);
  }
};

window.addEventListener("DOMContentLoaded", obj.hi);
```

Ooops!!! Turns out that since we added the event listener to the window object,
`this` in the event handler or callback refers to `window`.  So this code prints
`"Hi Bill"` instead of `"Hi Nick"`.  We could wrap `obj.hi` in an anonymous function:

```javascript
window.addEventListener("DOMContentLoaded", function () {
  obj.hi();
});
```

But that is so needlessly verbose and what we were trying to avoid in the first
place.  The three functions you should know for modifying `this` (a question I
ask all
my interview candidates) are
[Function.prototype.call](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/call),
[Function.prototype.apply](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/apply),
and
[Function.prototype.bind](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/bind).
`call` is variadic, while `apply` takes an array of
arguments, but the two both immediately invoke the function.  We don't want to
do that just yet.  The fix we need is `Function.prototype.bind`.

```javascript
window.addEventListener("DOMContentLoaded", obj.hi.bind(obj));
```

There, now isn't that nice and short?  Instead of saving `this` as another
variable then closing over it, you can instead use `bind`!

```javascript
var obj = {
  person: "Nick",
  wait: function () {
    var self = this;
    someButton.onclick = function () {
      console.log(self.person + " clicked!");
    };
  },
};
```
becomes
```javascript
var obj = {
  person: "Nick",
  wait: function () {
    someButton.onclick = function () {
      console.log(this.person + " clicked!");
    }.bind(this);
  },
};
```

No need to store `this` into `self`, then close over it.  One great shortcut I
use all the time is creating an alias for `document.getElementById`.

```javascript
var $ = document.getElementById.bind(document);
$('someElementsId').doSomething();
$('anotherElement').doSomethingElse();
$('aThirdElement').doSomethingDifferent();
$('theFifthElementOops').doSomethingFun();
```

Why did I bind `getElementById` back to `document`?  Try it without the call to
bind.  Any luck?

`bind` can also be great for partially applying functions, too.

```javascript
function add (a, b) {
  console.log("a: " + a);
  console.log("b: " + b);
  return a + b;
};
var todo = add.bind(null, 4);
console.log(todo(7));
```
will print
```
a: 4
b: 7
11
```

What `Function.prototype.bind` is essentially doing is wrapping `add` in a
function that essentially looks like:

```javascript
var todo = function () {
  add.apply(null, [4].concat(Array.prototype.slice.call(arguments)));
};
```

The array has the captured arguments (just `4`), and is converting `todo`'s
`arguments` into an array (a common idiom for converting "Array-like" objects
into
Arrays), then joining (`concat`) them and invoking the bound function (`apply`)
with
the value for `this` (in this case, `null`).

In fact, if you look at
[the compatibility section of the MDN page for bind](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/bind#Compatibility),
you'll see a function that returns a function that is essentially the above.
One caveat is that this approach only allows you to partially apply variables in
order.

So `bind` is a great addition to the language.  Now to the point I wanted to
make;
there are edge cases when `bind` doesn't work or might trip you up.  The first
is that `bind`
evaluates
its `arguments` when bound, not when invoked.  The other is that `bind` returns
a new
function, always.  And the final is to be careful binding to variadic functions
when you don't intend to use all of the passed in variables.  Um, duh right?
Well, let me show you three examples that have bitten me (recently).  The first
is with ajax calls.

```javascript
function crunch (data) {
  // operate on data
};

var xhr = new XMLHttpRequest;
xhr.open("GET", "data.json");
xhr.onload = crunch.bind(this.response);
xhr.send();
```

Oops, while I do want to operate on `this.result` within `crunch` with `this`
referring to `xhr`, `this` at the time of binding was referring to `window`!
Let's
hope `window.results` is `undefined`!  What if we changed `this.result` with
`xhr.result`?  Well, we're no longer referring to the `window` object, but
`xhr.result` is evaluated at bind time (and for an unsent `XMLHttpRequest`
object,
is `null`), so we've bound `null` as the first argument.  We must delay the
handling
of `xhr.onload`; either use an anonymous function inline or named function to
control nesting depth.

```javascript
xhr.onload = function () {
  crunch(this.result);
};
```

The next is that `bind` always returns a new function.  Dude, it says that in
the docs,
[RTFM](http://xkcd.com/293/).
Yeah I know, but this case still caught me.  When removing an event
listener, you need to supply the **same** handler function.  Example, a `once`
function:

```javascript
function todo () {
  document.removeEventListener("myCustomEvent", todo);
  console.log(this.person);
});

document.addEventListener("myCustomEvent", todo.bind({ person: "Nick" }));
```

Try firing `myCustomEvent` twice, see what happens!  `"Nick"` is logged twice.
A `once` function that handles two separate events is not very good.  In fact,
it will continue
to handle events, since `document` does not have `todo` as an event handler for
`myCustomEvent`
events.  The event listener you bound was a new function; `bind` always returns
a new function.  The solution:

```javascript
var todo = function () {
  console.log(this.person);
  document.removeEventListener("myCustomEvent", todo);
}.bind({ person: "Nick" });
document.addEventListener("myCustomEvent", todo);
```

That would be a good interview question.  The final gotcha is with functions
that are variadic.  Extending one of my earlier examples:
```javascript
var obj = {
  person: "Nick",
  wait: function () {
    var someButton = document.createElement("button");
    someButton.onclick = function () {
      console.log(this.person + " clicked!");
    }.bind(this);
    someButton.click();
  },
};
obj.wait();
```

Let's say I thought I could use bind to simplify the `onclick` using the trick I
did with `document.getElementById`:

```javascript
var obj = {
  person: "Nick",
  wait: function () {
    var someButton = document.createElement("button");
    someButton.onclick = console.log.bind(console, this.person + " clicked!");
    someButton.click();
  },
};
obj.wait();
```

Can you guess what this prints?  It does prints the expected, but with an
unexpected addition.  Think about what I said about variadic functions.  What
might be wrong here?

Turns out this prints
`"Nick clicked! [object MouseEvent]"`  This one took me a while to think
through, but luckily I had other experiences with `bind` that helped me understand
why this occurred.

`console.log` is variadic, so it prints all of its arguments.  When we called
`bind`
on `console.log`, we set the `onclick` handler to be a new function that applied
that expected output with any additional arguments.  Well, `onclick` handlers are
passed a `MouseEvent` object (think `e.target`), which winds up being passed as
the second
argument to `console.log`.  If this was the example with `add` from earlier,
`this.person + " clicked!"` would be the `4` and the `MouseEvent` would be the
`7`:

```javascript
someButton.onclick = function (e) {
  console.log.apply(console, ["Nick clicked!"].concat([e]));
};

```

I love `bind`, but sometimes, it will get you.  What are some examples of times
when you've been bitten by `bind`?
