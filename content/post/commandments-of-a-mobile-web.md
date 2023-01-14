+++
title = "Commandments of a Mobile Web"
date = "2013-02-28"
slug = "2013/02/28/commandments-of-a-mobile-web"
Categories = ["web"]
+++
Over the past few years, there's been certain
[paradigm shifts](http://en.wikipedia.org/wiki/Paradigm_shift)
in web development. When you think of milestones that really changed how
development on the web was done, the two biggest were
[Ajax](https://www.ibm.com/developerworks/mydeveloperworks/blogs/BobZurek/entry/the_ajax_paradigm_shift?lang=en)
and [HTML5](http://www.html5rocks.com/en/why).  Development was identifiably
different before and after such technological advancements.  There were
[some](http://java.sys-con.com/node/315210) who initially
[doubted](http://blog.tobie.me/post/31366970040/when-im-introspective-about-the-last-few-years-i)
the technologies, but I'm sure such doubters eventually
[saw the light](http://www.sencha.com/blog/the-making-of-fastbook-an-html5-love-story/).
After spending time working on applications for Mozilla's upcoming mobile
operating system, Firefox OS, and talking with my fellow employees, I feel that
the mobile web is another one of those shifts in how we
approach web development that looking back will be an identifiable point in
time where we can say that we did things differently before and after.
So in that sense, I want to share some of the insights I've found to
help other developers wrap their heads around how developing for the mobile web
isn't their traditional cup of tea.

# Internet connectivity is not guaranteed #
This is a fundamental divorce from the World Wide Web and the Internet. I feel
that a lot of people having trouble differentiating the Web from the Internet;
where you've had one, you've always had the other. Don't
assume your application will always have a valid connection.  When a user
is on a wifi connection, or hardwired, it's so obvious that if they're on your
website, then they must be connected to the Internet.  Right?  But what happens
now when one of your users loads up your site or app on a mobile device, then enters
a tunnel?  What does it do when offline?  Does it work?  Maybe it
doesn't make sense if you're offering a service that requires data from the
back end, but does that mean that the front end should totally look like crap
or break when the back end can't be reached?  Do your Ajax requests have error
callbacks?  Do you try and reconnect your WebSocket connections after they've
failed?  Do you cache previous requests or use one of the many forms of offline
storage to show something to the user?  Developing on the loopback is literally
the best case for connectivity, but it gives the developer a false sense of
how latency and connectivity issues affect their application.  It's literally
shocking the first time you encounter your site/application offline if you
didn't think about the user experience up front.

# Bandwidth is not free #
The advent of broadband made it acceptable for sites to utilize massive amounts
of assets.  Seeing sites that load up so much stuff makes my heart sink when
I wonder about viewing such a site on my mobile device.  I only pay for so much
mobile data per month, then I'm billed ridiculous amounts as a 
["Data Overage"](http://forums.att.com/t5/Data-Messaging-Features-Internet/What-does-quot-Data-MB-Overage-quot-mean/td-p/2872883).
Users in other countries frequently have a more pay as you go style plan, so
they're being billed for each bit across the wire.  In countries without the
infrastructure necessary to get broadband Internet to every household, mobile
is more prolific in getting users connected.

Things like minification and gzip certainly help, but deferred loading of
assets until they're necessary is frequently overlooked.
Massive libraries are nice for devs, but may end up giving mobile users more
problems than they are worth. There exist more advanced techniques such as
[WebSocket compression](http://buildnewgames.com/optimizing-websockets-bandwidth/).

How much does your site [weigh](http://www.sitepoint.com/minimizing-page-weight-matters/)?

# An expensive website does not make an awesome app #
{{< blockquote >}}
Why doesn't my million dollar website make a kick ass web app?
{{< /blockquote >}}

This point is brought to you by [Matt Basta](http://www.mattbasta.com/).  The
point is that there is something fundamentally different between a web "site"
and a web "app".  In a web site the usual flow involves loading up a page
and maybe navigating between pages.  An app more often than not will be a
single page that loads data asynchronously and dynamically modifies the content
displayed to the user.  Some web sites make great use of things like Ajax, and
are great example of single page sites.  But not all are.  Some still haven't
come around to loading all data asynchronously on request.  And it's not that
they necessarily even have to, they can be a website and never have to be
anything more.  But you'll find that some sites make better apps than others.

## No chrome ##
Not the browser made by Google, Google Chrome; the actual
[controls](https://developer.mozilla.org/en-US/docs/Chrome) on the top
of your browser such as the back and reload buttons, and the url bar.  This
point comes from [Chris Van Wiemeersch](https://github.com/cvan).  One of
the things you'll find when evaluating whether a site makes a good app is
whether it relies on chrome to navigate.  If it does, then it's not really
going to cut it as an app.  If you're going to make a single page app, try
making it fullscreen in your browser and try navigating around. Are you
hindered without the chrome?  One thing I
found recently was an error condition where I notified the user that they had
to be online to check for an update.  But when I was developing, I would just
refresh the page to clear the error message and try again.  On a device, as an
app, where there was no chrome, this meant closing and restarting the app.
That sucked, so I made it so the user could click/touch, after figuring out
their connectivity issues, to retry the fetch.  Even better may have
been to listen for online events!

## Traditional forms of input are not fun ##
Here's some tips from [Matthew "Potch" Claypotch](http://potch.me/).

### Fingers aren't as precise as cursors ###
Having tiny click
targets is really frustrating.  How many times have you clicked the wrong
thing on a mobile device?  Tiny buttons are not the easiest thing for users
to specify, especially when groups of them are clustered nearby.  Custom
buttons enabled by [a little CSS](http://www.cssbuttongenerator.com/) can go a
long way.

### Typing on little keyboards in tedious ###
This is an effect of tiny buttons, but requiring the user to type in large
strings gets annoying fast.  Sometimes this can't be avoided, but it should be
when it can.  Just as typing in a long, complex url to a mobile browser is
not enjoyable, neither is doing so in an app.

# Detect features not browser engine (User agent sniffing is a sin) #
Print [this](http://diveintohtml5.com/everything.html) and staple it to your
wall above your workspace.  This should be
[old](http://www.nczonline.net/blog/2009/12/29/feature-detection-is-not-browser-detection/)
[news](http://msdn.microsoft.com/en-us/magazine/hh475813.aspx) at this point.

## Vendor prefixes are meant for browser vendors to test, not production code ##
I place a majority of the blame for this on vendors; prefixes should never see
the light of day in release builds.  It's ridiculous to have to write four
copies of the same rule, when you're trying to express one thing.  I should
rewrite this sentence four different ways to prove a point.  -o-Do you understand
what I'm getting at?  -ms-It's almost like rambling. -moz-Repetitive department
of repetition. -webkit-This is ridiculous.

But developers need to recognize the habit as an addiction, and not feed it.
If you have, I forgive you.  Now stop doing it.

These two particular articles are just so good, it wouldn't do them justice to
try and summarize them.  Please go read them, I'll wait.
[this](http://hsivonen.iki.fi/vendor-prefixes/)
and
[this](http://www.quirksmode.org/blog/archives/2010/03/css_vendor_pref.html)

## Developing towards WebKit and not HTML5 is a sin ##
I understand that Google Chrome is your favorite browser, and I am so happy for
you; but it is not mine.  I'll be the first to admit that Google
caught all of the other browser vendors with their pants down, but when I see
pages that look flawless in Chrome and
not so hot in others, it reminds me of days when sites only worked in IE6.
Surely you remember *those* days.  I hate when content publishers try and
dictate which browser I should use to view their content.  I understand WebKit
based browsers dominant in mobile, but so did IE6 in desktop share at one point.
It's not unreasonable to expect users to use the most updated version of their
browser, but empower your users to choose their browser.  Don't take that
choice away from them.

A neat [point](http://robertnyman.com/2013/02/14/webkit-an-objective-view/) by
[Robert Nyman](http://robertnyman.com/) is that WebKit itself already has forks.
Can you imagine if there were eventually vendor prefixes for forks of WebKit?
Continuing with vendor prefixes means that we'll now have seven vendor prefixes:
unprefixed, -moz-, -o-, -ms-, -webkit-o-, -webkit-chrome-, -webkit-safari-.
Awesome!  Maybe I should rewrite this sentence seven different ways to make a
point!

I'm also curious if Google, Apple, and the WebKit maintainers are turning a
blind eye to this, or what their opinions are? Being a vendor and wanting an
open web are almost conflicts of interest; you want your browser to "win" or
dominate in marketshare, but at the same time you don't want any one browser
having too much marketshare.

# Design Responsively #
What is
[responsive design](http://mashable.com/2012/12/11/responsive-web-design/)?
[Responsive design](http://www.smashingmagazine.com/responsive-web-design-guidelines-tutorials/)
is making a site look great on
any size screen, without duplicating assets or sniffing a user agent string.
Firefox has a neat web dev tool called
["Responsive Design View"](https://developer.mozilla.org/en-US/docs/Tools/Responsive_Design_View).
It's great for testing out your site on various screen sizes. 
[Twitter Bootstrap](http://twitter.github.com/bootstrap/index.html) is an
excellent example of a framework for developing a single app that looks great
on any screen size.  Even if you don't want to use a whole big framework,
simple things like using CSS rules in terms of percentages instead of hard
coded pixels can go a long way.

Sites that are trying to become more app like have trouble with conforming to
responsive design.  Instead of starting with a site and trying to figure out
how to hide
or not display information as the screen gets smaller, you'll find it much
[easier](http://johnpolacek.github.com/scrolldeck.js/decks/responsive/) to
start small, and dynamically add content as the screen gets bigger.

# High performance code respects battery life #
In the end of the day, all of the code you write that runs in the browser is
a set of instructions the processor can decode, not necessarily the high level
JavaScript you wrote.  While it's important to avoid premature optimizations,
having a few
[rules of thumb](http://christianheilmann.com/2013/01/25/five-things-you-can-do-to-make-html5-perform-better/)
up front will help you write better, faster code.
If the same overall action can be expressed in one instruction or one hundred,
which do you think will use
[less power](http://coding.smashingmagazine.com/2012/11/05/writing-fast-memory-efficient-javascript/)?
Keep in mind that transistors leak current while switching.

## Native methods over library methods, CSS over JS ##
What do I mean by "native methods?" Like C++?  Well, yes.  You see under the
hood the DOM bindings that you're calling are probably written in C++.  Call
`toSource()` on `document.getElementById()`.  The `[native code]` statement
in the returned string refers to the implementation.  While type specialized
code emitted by the JIT can match or even beat native code, you can only count
on that for hot loops.  In the same vein, library code is going to be
[slower](http://jsperf.com/getelementbyid-vs-jquery-id/25) than any code
written natively.  I'm not saying that you shouldn't use libraries, just know
that the use of libraries can incur some overhead.  Things like animations will
also be faster
when handled by natively implemented CSS over JS.  You can use JS to
dynamically add classes to elements to get finer resolution over events, but
then get the performance of CSS.

## Avoid JIT bailout ##
The Just In Time (JIT) interpreter is a
[new breed](http://www.slideshare.net/amdgigabyte/know-your-javascript-engine)
of VM that most browser
vendors are now using.  The JIT can
[analyze](http://s3.mrale.ph/nodecamp.eu/#1) running code, and
[emit](http://en.wikipedia.org/wiki/Code_generation_%28compiler%29#Runtime_code_generation)
native code
that is faster than reinterpreting code again, and even higher-optimized code
for type stable JavaScript, as long as certain "guard" conditions are met.
When a guard fails, the JIT has to
[bailout](http://mxr.mozilla.org/mozilla-central/source/js/src/ion/Bailouts.h#19)
the emitted native code and start reinterpreting code again.

## Keep up to date on new APIs ##
HTML5 is a big
[spec](http://www.w3.org/html/wg/drafts/html/master/single-page.html), and is
getting bigger.  So big that some recommendations are
being [spun off](http://www.websocket.org/aboutwebsocket.html) from the
original HTML5 spec.  As an engineer, I'm painfully aware that you need to
[keep up](http://bonsaiden.github.com/JavaScript-Garden/)
in whatever industry you work in in order to stay relevant.
The complacent are the most vulnerable.  There's
[a lot to keep track of](http://caniuse.com/) with
HTML5 and CSS3, but many new features offer higher performance methods of
skinning the cat.

### requestAnimationFrame ###
window.requestAnimation frame is a godsend for animation.  Not too long ago, I
wrote up a quick
[example](https://github.com/nickdesaulniers/canvas2dcontext/blob/master/examples/sprite.html#L6)
of various ways of implementing animation loops and their issues; you should
check it out.

### indexedDB over localstorage ###
The indexedDB api might not be as simple as localstorage's is, but localstorage
is synchronous and is noticeably slow on mobile.  If you can bite the bullet
and use indexedDB, you'll find you're JS isn't blocking on
serializing/deserializing objects for storage.
[Fabrice Desré](https://twitter.com/fabricedesre) shared this with me.

### WebWorkers ###
Webworkers can't modify the DOM, but they can do heavy lifting without blocking
the main thread.

### CSS translate over absolute top and left rules ###
[Harald Kirschner](http://digitarald.de/) recommends CSS translates over top
and left rules for absolutely positioning some elements.

### Gradients are expensive ###
[Dan Buchner](http://www.backalleycoder.com/) notes that without beefy graphics
processing units of their desktop counterparts to enable hardware acceleration,
things like gradients will give you noticeable performance hits.

### createDocumentFragment ###
Dan also suggests queuing up DOM changes.
Whenever you manipulate items in the DOM, you're going to trigger a reflow,
which may consist of an update to the layout and/or a repaint.  Minimizing these
makes for a faster update to the DOM.  For example, it can be faster to use
document.createDocumentFragment and append child nodes to it, and then append
that to the DOM, instead of appending lots of child nodes in between timer calls.
Surprise, this isn't actually a
[new](http://ejohn.org/blog/dom-documentfragments/) DOM binding from HTML5.

# Conclusion #
These are just some tips I have for application developers.  I am by no means
an expert; I'm sure if you dig deep enough, you can find plenty of examples of
my past work that contradicts some of my recommendations from this article.  But
I'm a little smarter today than I was yesterday, and now so are you!  What are some
tips that you have to share that you've found helpful developing for the
mobile web?
