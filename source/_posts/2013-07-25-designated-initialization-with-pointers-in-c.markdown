---
layout: post
title: "Designated initialization with compound literals in C"
date: 2013-07-25 16:48
comments: true
categories: [C, initialization, designated, compound, literal, nested, struct]
published: true
---

Just a quick post on something I just discovered and found neat (I always find
obscure C syntax interesting).  I was trying to figure out how to use a C
designated initializer, where a member was a pointer to another designated
initializer.  At this point, you need a compound literal.  Just a quick
background on C initialization:

```c
// verbosely create an array with a known size
int arr [3];
arr[0] = 1;
arr[1] = 2;
arr[2] = 3;
// => [1, 2, 3]

// concisely create an array with a known size
int arr [3] = { 1, 2, 3 }; // => [1, 2, 3]

// creates an array with unspecified values initialized to 0
int arr [4] = { 1, 2, 3 }; // => [1, 2, 3, 0]

// truncates declaration
int arr [1] = { 1, 2, 3 }; // => [1]

// based on number of initializers
int arr [] = { 1, 2, 3 }; // => [1, 2, 3]
```

Let's look at how we might have initialized a struct in C89.  In C89, you are
required to declare local variables at the top of a block.  A previous
initialization of a point struct might have looked like:

```c
struct point {
  int x, y;
};

{
  struct point a;
  a.x = 2;
  a.y = 3;
}
```

Just as we can define array literals in C using the initializer list syntax, we
can use the same concise syntax for initializing structs!

```c
// point a is located at (2, 3)
struct point a = { 2, 3 };
```

Well, this can be bad.  Where would point a be located if say a fellow team
mate came along and modified the definition of the point struct to:

```c
struct point {
  int y, x; // used to be `int x, y;`
};
```

Suddenly point a points to (3, 2), not (2, 3).  It's better if you use
designated initializers to declare the values for members of your struct.  It's
up to the compiler to decide on the order of initialization, but it wont mess
up where the data is intended to go.

```c
// point b is located at (2, 3)
struct point b = { .y = 3, .x = 2 };
```

So now we have designated initializers, cool.  What about if we want to use the
same syntax to reassign point b?

```c
b = { .x = 5, .y = 6 };
//    ^
// error: expected expression
```

While you are being explicit about the shape of the struct that you are trying
to assign to b, the compiler cannot figure out that you're trying to assign a
point struct to another point struct.  A C cast would probably help here and
that's what the concept of compound literals are.

```c
b = (struct point) { .x = 5, .y = 6 }; // works!
```

Notice: I just combined a compound literal with a designated initializer.  A
compound literal on its own would look like:

```c
b = (struct point) { 5, 6 }; // works!
```

To recap we can define points like so:

```c
struct point a;
a.x = 1;
a.y = 2; // C89 (too verbose)
struct point b = { 3, 4 }; // initializer list (struct member order specific)
struct point c = { .x = 5, .y = 6 }; // designated initializer (non struct member order specific)
struct point d = (struct point) { .x = 7, .y = 8 }; // compound literal (cast + designated initialization)
```

My favorite part of compound literals is that you can define values inline of
an argument list.  Say you have a function prototype like:

```c
int distance (struct point, struct point);

// instead of calling it like this
// (creating temp objects just to pass in)
struct point a = { .x = 1, .y = 2 };
struct point b = { .x = 5, .y = 6 };
distance(a, b);

// we can use compound literals
distance((struct point) { .x = 1, .y = 2 }, (struct point) { .x = 5, .y = 6 });
```

So compound literals help with reassignment of structs, and not storing
temporary variables just to pass as function arguments.  What happens though
when one of your members is a pointer?  C strings are easy because they already
have a literal value:

```c
struct node {
  char *value;
  struct node *next;
};

// just using designated initialization
struct node a = {
  .value = “hello world”,
  .next = NULL
};
```

But what happens if we want to initialize node.next?  We could do:

```c
struct node b = {
  .value = “foo”,
  .next = NULL
};
a.next = &b;
```

Again, we have to define b before assigning it to a.next.  That's worthwhile if
you need to reference b later in that scope, but sometimes you don't (just like
how compound literals can help with function arguments)!  But that's where I
was stumped.  How do you nest designated initializers when the a member is a
pointer to another designated initializer?  A first naïve attempt was:

```c
struct node c = {
  .value = “bar”,
  .next = {
    .value = “baz”,
//  ^
// error: designator in initializer for scalar type 'struct node *'
    .next = NULL
  }
};
```

WTF?  Well, if you go back to the example with nodes a and b, we don't assign
the value of b to a.next, we assign it a pointer to b.  So how can we use
designated initializers to define, say, the first two nodes of a linked list?
Compound literals.  Remember, a compound literal is essentially a designated
initialization + cast.

```c
struct node d = {
  .value = “qux”,
  .next = &((struct node) {
    .value = “fred”,
    .next = NULL
  })
};
```

And that works, but why?  d.next is assigned an address of a
compound literal.  Granted, you probably don't want to be declaring your entire
linked list like this, as nesting gets out of control fast.  I really like this
style because it reminds me of JavaScript's syntax for declaring object
literals.  It would look nicer if all of your nested structs were values and
not references though; then you could just use designated initializers and
wouldn't need compound literals or address of operators.

What's your favorite or most interesting part of C syntax?

Acknowledgements:

* [Louis Stowasser](http://louisstowasser.com/)
* [Frederic Wenzel](http://fredericiana.com/)
* [21st Century C](http://shop.oreilly.com/product/0636920025108.do)

