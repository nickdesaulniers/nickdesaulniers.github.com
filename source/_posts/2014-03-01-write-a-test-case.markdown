---
layout: post
title: "Write a Test Case"
date: 2014-03-01 13:22
comments: true
categories: [Test, Case, Bug, Issue, Report, Technical, Developer, Open, Source, Tracker]
published: true
---
Your application just broke, oh no!  It couldn't have been *your* code, right?

I've always had trouble spotting mistakes in my own work such as spelling,
grammar, mathematical, or even in programming.  With spelling or grammar,
office applications quickly pick up on my mistakes and underline them for me,
but most of my mistakes come from my own hubris.  I'm confident in what I do,
and that gets me in trouble when I make little mistakes.  I'm confident that I
solved this problem correctly and didn't make any small mistakes.  As a kid
competing in timed math competitions, I quickly learned that reviewing your
work cost time, so it was important to recognize where common mistakes would
crop up on certain problems and spend slightly extra time the first time
through those problem areas, rather than double checking the work in its
entirety, unless I had the time to spare.  Measure twice, cut once.

The same is true for programming.  Writing test cases is time consuming, and if
time is money, then it could be said that writing tests is costly.  There's
probably a logical fallacy in there.  In general, we hope that the time-cost of
writing test
cases will be recuperated by the time-cost of bugs caught, though it's not easy
to measure the time-cost of averted bugs.  I'm all for test cases.  I think
that
[SQLite having 100% branch coverage](http://sqlite.org/testing.html#coverage)
is incredible, truly an lofty
achievement.  People get bogged down in arguments over type systems where
testing is more of a must for languages without a compiler to catch certain
bugs.

Ok, so going back to your code, something is wrong.  But we're confident it
couldn't be *our* code.  Maybe it's one of those open source modules I'm using
not being tested enough, or that pesky browser implementation, or this damned
OS.  It couldn't be *my* code.

**Prove it.**

I see bug reports all the time where people say your X breaks my Y, but when
asked to provide Y they can't for whatever software licensing reason.  The bug
resolver (person who is enabled to fix said bug in X), doesn't know at this
point whether the bug reporter (developer of Y) did their homework; the bug is
unconfirmed.  Both reporter and resolver are suspicious of each others' code.
*I* don't make silly mistakes, right?

{% img images/dude.jpg %}

So it's up to the resolver to work out a test case.  Being able to reproduce
said issue is the first goal to resolving a bug.  Just like scientific
statements aren't taken as fact until reproducible, so too will this bug be
merely conjecture at this point.  It kills me when the bug resolver closes an
issue because it *works for me*.  Nothing boils my blood more; I took the time
to try and help you and wasn't rewarded on some level so I feel that I wasted
my time.  But from where you stand the resolver is not correct, so again I
say...

**Prove it.**

I think that it's super hard to get meaningful feedback from users.  It's hard
for them to express why they're frustrated.  We as software developers don't
always point them in the right direction.  As users can have wide ranges of
technical capabilities, sometimes we as more technical oriented people have to
do a little hand holding.  For instance, from Firefox, how many hops does it
take to get from the application's menus to their open bug tracker,
[bugzilla](https://bugzilla.mozilla.org/)?
Where can I report an issue?  How do I report an issue?
Do I annoy the hell out of the user until they rate my app?
Users might be complaining on Twitter, but that's sure as hell not where
devlopers are looking for bug reports.
Enabling the user to provide feedback could be
viewed as a double edged sword.  Can I make meaningful conclusions from the
torrent of feedback I'm now getting?  Did I structure my form in such a way to
ask the right questions?  Should we make our issue tracker public?  Should we
make security issues that could harm users, if more widely understood, public?
Public to fellow employees, even?  What did you expect to happen, and is that
what actually happened?  This rabbit hole goes deep.

The other day, I was writing a patch for a large application that was developed
by someone else and that I still don't fully comprehend in its entirety. My
patch should have worked, why wasn't it working?  Probably something in this
person's code right?  Sure enough, I wrote a simple test case to eliminate
everything but the control variables, and it turns out *I was wrong*.

It's hard to get users to
do more than the bare minimum to report an issue, even at all, but to prevent
the issue from being ignored or closed because it works for the developer in
their environment, take the time to report it correctly the first time.
As someone who understands technology more than the average person, take the
time to write a cut down test case that
exposes the issue. The developer who looks at the issue will be grateful;
instead of wasting their time with something that may not even be a bug, you've
just done something that would have to be done anyways if it is indeed a bug.
You might find that a reduced code size shows that it's in
fact not an issue with someone else's code, but in fact a simple mistake you
made somewhere in your monolithic application.  Or it might be exactly the
evidence you need to get someone's attention to fix said problem.  That test
case might even be included in the latest test suite.  Either way, you have now
proved beyond a reasonable doubt that the issue does not lie in your fault.

**Write a Test Case for Your Next Bug Report or Issue.**

