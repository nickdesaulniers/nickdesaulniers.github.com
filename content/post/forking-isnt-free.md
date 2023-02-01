---
title: "Forking is not free; the hidden costs"
date: 2023-02-01T13:42:57-08:00
categories:
- Open Source Software
- security
---
The Open Source software development model is a powerful approach to
collaboratively iterating on a shared solution to a common problem.  Rather
than competitors duplicating effort to come up with multiple suboptimal
solutions, they can collaborate together to build something greater.

Forking is the term used to denote making a copy of a codebase. The copy, known
as a "fork" is used as a starting point for work that may be contributed back
to the original maintainers; the canonical "upstream" from which we derive our
"downstream" clone.  These copies share a similar origin point; think of all of
the things in life that share a similar origin, but due to various factors have
diverged slightly over time in assorted measures.

Other times and for various reasons, a fork is used to diverge from upstream
(intentionally or unintentionally).

- Have ideas for improvements? Fork it!
- Need a point in time to stabilize and de-risk a release? Fork it!
- Dead project no longer maintained? Fork it!
- Create your own backup for archival purposes? Fork it!
- Release cadence, or development process doesn't work for you? Fork it!
- Disagree with the current maintainers? Fork it!

It's so easy, GitHub has a big button right in their UI encouraging you to
fork.  What are you waiting for?  While the marginal cost of duplicating
software is practically zero, the actual cost over time of *maintaining* a fork
can be much greater than zero.  Maintenance costs of software are perhaps less
well understood than marginal costs.

The process of updating a fork is known as "rebasing," re-establishing the
point at which a fork first diverged.  Depending on how much work was done on
the mainline vs the fork (imagine the distance between two vectors with the
same origin), the work involved in rebasing will be directly proportional to
the distance of the divergence.  Depending on what was modified, and by how
much, rebasing can be painless.  Or it can take years (yes, I've seen it),
taking resources away from development of new features and improvements and bug
fixing.

It can also harm your ability to respond to security vulnerabilities.  It sucks
when you're not agile enough to deploy zero day fixes quickly because backports
are infeasible, and yet moving forward is also infeasible. Ask me how I know.
Having a vehicle for rapid release (even if it's not used continuously) at
least allows you to have a plan for such black swan events.

In this way, developing software from a fork is a source of accumulation of
tech debt; it may not cost you anything today.  But should you ever plan to
rebase your software, *you will pay for it*.

If you plan on rebasing your fork ever, consider minimizing your cost of
rebasing your divergence by pushing your changes upstream as a means of
avoiding the accumulation of technical debt. This involves compromise,
persistence, and patience. You might not think you have time to do so today,
but if you ever need to rebase in the future, pray for the soul of whoever gets
that assignment.

I think it's fun to imagine what factors might influence a cost function
measuring the costs of forking.

Cost = R * (E + D * U * C)

Where:
- R = number of rebases planned
- E = loss of expertise over time
- D = number of downstream changes
- U = number of upstream changes
- C = conflict factor between upstream and downstream

The number of rebases planned is clearly a factor. Given a timespan and planned
frequency of rebasing, you can calculate the number of rebases in a time frame.
If no rebases are planned, the cost goes to zero (i.e. parking a fork, cutting
a release). More rebases means more pain.

Merge conflicts can be painful to have to resolve. How painful is a factor of
the number of upstream vs downstream changes.  Sometimes there's no conflict,
so I imagine there's some constant factor between common lines modified.
Sometimes code rebases cleanly, but behaviors have changed.  That can result in
time consuming and surprising bugs or test failures that need to be resolved.
If the number of upstream changes is zero, there's nothing to rebase onto. If
the number of downstream changes is zero, you're simply updating your snapshot.
I'd like to think that such a case would be zero cost, but validation costs are
something I omitted from the above model. Validation costs can add
significantly to rebases where no downstream changes are involved!

Something that's not a large factor, but something I've definitely experienced
before is when the authors of downstream changes aren't active in the project
anymore and have moved on (like when you've been carrying downstream changes
for years).  When there's conflicts or tests start failing due to change in
behavior, knowing precisely what the intent of downstream changes were when
authored can be difficult. Lately I find myself including comments in tests
about what my intent was, just in case the code itself isn't obvious enough.
This might not even be dependent on the number of downstream changes. If you're
rebasing a codebase you're unfamiliar with and automated tests start failing,
do you even know how to rerun the tests manually or start debugging them? Maybe
you *had* a teammate that did.

What other factors can you think of increase or reduce the costs of forking?
Are the factors I imagine above the correctly weighted? I feel like something
there should be exponential and not multiplicative...

All in all, these factors should be considered when forking a project. Seeing
teams carry large out of tree patch sets and then struggle with rebasing is
disappointing and makes me wonder if they (or their management) perhaps
underestimated the maintenance costs of creating a fork.

Your fork might not cost you anything today, but if you ever plan to rebase
your out of tree changes, *you're going to pay for it*.  I encourage you to get
those patches upstream, and wipe away the [tech] debt!
