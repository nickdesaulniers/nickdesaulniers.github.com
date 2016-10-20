---
layout: post
title: "Writing my first technical book chapter"
date: 2015-01-25 20:50
comments: true
categories: [WebGL, Emscripten, Asm.js, Web, GL, Insights, Author, Graphics]
published: true
---
It's a feeling of immense satisfaction when we complete a major achievement.
Being able to say "it's done" is such a great stress relief.  Recently, I
completed work on my first publication, a chapter about Emscripten for the
upcoming book
[WebGL Insights](http://www.crcpress.com/product/isbn/9781498716079)
to be published by CRC Press in time for
[SIGGRAPH 2015](http://s2015.siggraph.org/).

One of the life goals I've had for a while is writing a book.  A romantic idea
it seems to have your ideas transcribed to a medium that will outlast your
bones.  It's enamoring to hold books from long dead authors, and see that their
ideas are still valid and powerful.  Being able to write a book, in my eyes,
provides some form of life after death.  Though, one could imagine ancestors
reading blog posts from long dead relatives via utilities like the
[Internet Archive's WayBack Machine](https://web.archive.org/web/20141218200253/http://nickdesaulniers.github.io/).

Writing about highly technical content places an upper limit on the usefulness
of the content, and shows as "dated" quickly.  A book I recently ordered was
[Scott Meyers' Effective Modern C++](http://shop.oreilly.com/product/0636920033707.do).
This title strikes me, because what
exactly do we consider *modern* or *contemporary*?  Those adjectives only make
sense in a time limited context.  When C++ undergoes another revolution,
Scott's book may become irrelevant, at which point the adjective *modern*
becomes incorrect.  Not that I think Scott's book or my own is time-limited in
usefulness; more that technical books' duration of usefulness is significantly
less than philosophical works like *1984* or *Brave New World*.  Almost like having
a record in a sport is a feather in one's cap, until the next best thing comes
along and you're forgotten to time.

Somewhat short of my goal of writing an entire book, I only wrote a single
chapter for a book.  It's interesting to see that a lot of graphics programming
books seem to follow the format of one author per chapter or at least multiple
authors.  Such book series as *GPU Gems*, *Shader X*, and *GPU Pro* follow this
pattern, which is interesting.  After seeing how much work goes into one
chapter, I think I'm content with not writing an entire book, though I may
revisit that decision later in life.

How did this all get started?  I had followed Graham Sellers on Twitter and saw
[a tweet from him](http://octopress.org/2015/01/15/octopress-3.0-is-coming/)
about a call to authors for WebGL Insights.  Explicitly in the linked to page
under the call for authors was interest in proposals about Emscripten and
asm.js.

[Tweet](https://twitter.com/grahamsellers/status/504974663848456193)

At the time, I was headlong into a project helping Disney port Where's My Water
from C++ to JavaScript using Emscripten.  I was intimately familiar with
Emscripten, having been trained by one of its most prolific contributors,
[Jukka Jylänki](http://clb.demon.fi/).
Also, Emscripten's creator,
[Alon Zakai](http://mozakai.blogspot.com/), sat on the other side
of the office from me, so I was constantly pestering him about how to do
different things with Emscripten.  The #emscripten irc channel on
irc.mozilla.org is very active, but there's no substitute for being able to
have a second pair of eyes look over your shoulder when something is going
wrong.

Knowing Emscripten's strengths and limitations, seeing interest in the subject
I knew a bit about (but wouldn't consider myself an expert in), and having the
goal of writing something to be published in book form, this was my opportunity
to seize.

I wrote up a quick proposal with a few figures about why Emscripten was
important and how it worked, and sent it off with fingers crossed.  Initially,
I was overjoyed to learn when my proposal was accepted, but then there was a
slow realization that I had a lot of work to do.  The editor,
[Patrick Cozzi](http://www.seas.upenn.edu/~pcozzi/), set up
[a GitHub repo](https://github.com/WebGLInsights/WebGLInsights-1)
for our additional code and figures, a mailing
list, and sent us a chapter template document detailing the process.  We had 6
weeks to write the rough draft, then 6 weeks to work with reviewers to get the
chapter done.  The chapter was written as a Google Doc, so that we could have
explicit control over who we shared the document with, and what kinds of
editing power they had over the document.  I think this approach worked well.

I had most of the content written by week 2.  This was surprising to me,
because I'm a heavy procrastinator.  The only issue was that the number of
pages I wrote was double the allowed amount; way over page count.  I was
worried about the amount of content, but told myself to try not to be attached
to the content, just as you shouldn't stay attached with your code.

I took the additional 4 weeks I had left to finish the rough draft to invite
some of my friends and coworkers to provide feedback.  It's useful to have a
short list of people who have ever offered to help in this regard or owe you
one.  You'll also want a diverse team of reviewers that are either close to the
subject matter, or approaching it as new information.  This allows you to stay
technically correct, while not presuming your readers know everything that you
do.

The strategy worked out well; some of the content I had initially written about
how JavaScript VMs and JITs speculate types was straight up wrong.  While it
played nicely into the narrative I was weaving, someone more well versed in
JavaScript virtual machines would be able to call BS on my work.  The reviewers
who weren't as close to subject matter were able to point out when logical
progressions did not follow.

Fear of being publicly corrected prevents a lot of people from blogging or
contributing to open source.  It's important to not stay attached to your work,
especially when you need to make cuts.  When push came to shove, I did have
difficulty removing sections.

Lets say you have three sequential sections: A, B, & C.  If section A and
section B both set up section C, and someone tells you section B has to go, it
can be difficult to cut section B because as the author you may think it's
really important to include B for the lead into C.  My recommendation is sum up
the most important idea from section B and add it to the end of section A.

For the last six weeks, the editor, some invited third parties, and other
authors reviewed my chapter.  It was great that others even followed along and
pointed out when I was making assumptions based on specific compiler or
browser.
[Eric Haines](http://erich.realtimerendering.com/) even reviewed my chapter!
That was definitely a highlight for me.

We used a Google Sheet to keep track of the state of reviews.  Reviewers were
able to comment on sections of the chapter.  What was nice was that you were
able to keep using the comment as a thread, responding directly to a
criticism.  What didn't work so well was then once you edited that line, the
comment and thus the thread was lost.

Once everything was done, we zipped up the assets to be used as figures,
submitted bios, and wrote a tips and tricks section.  Now, it's just a long
waiting game until the book is published.

As far as dealing with the publisher, I didn't have much interaction.  Since
the book was assembled by a dedicated editor, Patrick did most of the leg work.
I only asked that what royalties I would receive be donated to Mozilla, which
the publisher said would be too small (est $250) to be worth the paperwork.  It
would be against my advice if you were thinking of writing a technical book for
the sole reason of monetary benefit.  I'm excited to be receiving a hard cover
copy of the book when it's published.  I'll also have to see if I can find my
way to SIGGRAPH this year; I'd love to meet my fellow authors in person and
potential readers.  Just seeing the list of authors was really a who's-who of
folks doing cool WebGL stuff.

If you're interested in learning more about working with Emscripten, asm.js,
and WebGL, I sugguest you pick up a copy of WebGL Insights in August when it's
published.  A big thank you to my reviewers: Eric Haines, Havi Hoffman,
Jukka Jylänki, Chris Mills, Traian Stanev, Luke Wagner, and Alon Zakai.

So that was a little bit about my first experience with authorship.  I'd be
happy to follow up with any further questions you might have for me.  Let me
know in the comments below, on Twitter, HN, or wherever and I'll probably find
it!

