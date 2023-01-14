+++
title = "C Function Pointers Alternate Syntax"
date = "2013-01-26"
slug = "2013/01/26/c-function-pointers-alternate-syntax"
Categories = ["C", "function pointer", "syntax"]
+++

On an interview with [Square](https://squareup.com/), I made the mistake
of stating that one of the benefits of working with JavaScript over C is
that functions are [first class](http://en.wikipedia.org/wiki/First-class_function)
in JavaScript, therefore they may be [passed around](http://eloquentjavascript.net/chapter6.html).  To which the
interviewer replied, "Well, C can do that, what about function pointers?"  What? Luckily, I
was able to get out of that jam by guessing that JavaScript had a nicer
syntax.

While I was taught some C in university, we had never gone over function
pointers or more in depth topics such as [static](http://en.wikipedia.org/wiki/Static_library)
or [dynamic](http://en.wikipedia.org/wiki/Dynamic_library) [linkage](http://www.yolinux.com/TUTORIALS/LibraryArchives-StaticAndDynamic.html).
I was so embarrassed that my expensive (read [overpriced](http://online.wsj.com/article/SB10001424127887324442304578231922159602676.html?mod=WSJ_hps_LEFTTopStories))
degree had not taught me more about
the C programming language, especially from the Computer Engineering
department that focuses on software __AND__ hardware.  On my exit
interview with the dean, I was very opinionated on the amount of C that
was (or wasn't) taught at my university.  His argument was that there
are only so much of so many languages you can cover in a university, which
to some extent is valid.  My problem has been that I've enjoyed learning
many different programming languages, though I didn't really get it the
first time around (with Java).  I think knowing many different languages
and their respective paradigms makes you a better programmer in other
languages, since you can bring a [Ruby Way](http://therubyway.org/) to
[Rust](http://www.rust-lang.org/) or a [JavaScript functional](http://www.ibm.com/developerworks/library/wa-javascript/index.html)
[style](http://interglacial.com/hoj/hoj.html) into C.

I had read two books in the meantime that really flushed out more of the
C language to me and I would definitely recommend them to those who want
to learn more about the language. They are [Head First C](http://shop.oreilly.com/product/0636920015482.do) by Dave and Dawn
Griffiths and [21st Century C](http://shop.oreilly.com/product/0636920025108.do) by Ben Klemens.
I'm also aware that [The C Programming Language](http://en.wikipedia.org/wiki/The_C_Programming_Language) by
Brian Kernighan and Dennis Ritchie is also known as the canonical text.
The book is often referred to as 'K&R' after the authors' initials, and Dennis
Ritchie was the original creators of the C language and co-developer of
Unix.  I'd love to get a chance to read it someday.


[This article](http://blog.charlescary.com/?p=95) appeared on
[Hacker News](http://news.ycombinator.com/) and really piqued my
interest.  Defenitely a great read.  What really stuck out to me was one
of the [comments](http://blog.charlescary.com/?p=95#comment-31) though.
The author of the comment mentioned a less ugly syntax for function
pointers, with a [link to an example](http://pastebin.com/MsJLY22j).
Now I'm not sure what the commenter meant by "these params decay
naturally to function pointers" but I was skeptical about this different
syntax.  Even the [Wikipedia article](http://en.wikipedia.org/wiki/Function_pointer#Example_in_C)
used the syntax that I was familiar with.  So I wrote up a quick example
to try it:

```c
// compiled with:
// clang -Wall -Wextra function_pointers.c -o function_pointers
#include "stdio.h"

void usual_syntax (void (*fn) (int x)) {
  puts("usual syntax start");
  fn(1);
  puts("usual syntax end");
}

void other_syntax (void fn (int x)) {
  puts("other syntax start");
  fn(2);
  puts("other syntax end");
}

void hello (int x) {
  printf("hello %d\n", x);
}

int main () {
  puts("hello world");
  usual_syntax(hello);
  other_syntax(hello);
}
```

Sure enough we get:

```
hello world
usual syntax start
hello 1
usual syntax end
other syntax start
hello 2
other syntax end
```

So the moral to the story I guess is that there's always more to your
favorite language.  From using [variadic macros](https://github.com/nickdesaulniers/21stCenturyC/blob/master/chpt10/variadic_macros.c)
with [compound literals](https://github.com/nickdesaulniers/21stCenturyC/blob/master/chpt10/compound_literal.c)
to enable a more [functional style](https://github.com/nickdesaulniers/21stCenturyC/blob/master/chpt10/foreach.c)
in C to reflecting upon a function's [number of arguments](https://developer.mozilla.org/en-US/docs/JavaScript/Reference/Global_Objects/Function/length)
or [name](https://developer.mozilla.org/en-US/docs/JavaScript/Reference/Global_Objects/Function/name)
or [adding attributes to a function](http://taylodl.wordpress.com/2012/09/03/functional-javascript-memoization-part-ii/)
or the
[y-combinator](http://matt.might.net/articles/implementation-of-recursive-fixed-point-y-combinator-in-javascript-for-memoization/)
in JavaScript, I learn something new every day.  And I hope that you did
too!  Thanks for reading!  If you have some other recommendations on
good programming books, or design patterns, please leave a comment or
write a reply blog post!
