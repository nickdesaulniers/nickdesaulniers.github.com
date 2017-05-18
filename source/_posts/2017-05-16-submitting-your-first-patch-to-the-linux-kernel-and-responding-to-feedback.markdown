---
layout: post
title: "Submitting Your First Patch to the Linux kernel and Responding to Feedback"
date: 2017-05-16 01:02
comments: true
categories: Linux Kernel Patch First Feedback Respond Email
---
After working on the Linux kernel for Nexus and Pixel phones for nearly a year,
and messing around with the
excellent [Eudyptula challenge](http://eudyptula-challenge.org/), I finally
wanted to take a crack at submitting patches upstream to the Linux kernel.

This post is woefully inadequate compared to the existing documentation, which
should be preferred.

* http://elixir.free-electrons.com/linux/latest/source/Documentation/process
* https://kernelnewbies.org/FirstKernelPatch

I figure I’d document my workflow, now that I’ve gotten a few patches accepted
(and so I can refer to this post rather than my shell history...).  Feedback
welcome
([open an issue](https://github.com/nickdesaulniers/nickdesaulniers.github.com/issues) 
or email me).

## Step 1: Setting up an email client

I mostly use `git send-email` for sending patch files.  In my `~/,gitconfig` I
have added:

```ini
[sendemail]
  ; setup for using git send-email; prompts for password
  smtpuser = myemailaddr@gmail.com
  smtpserver = smtp.googlemail.com
  smtpencryption = tls
  smtpserverport = 587
```

To send patches through my gmail account.  I don’t add my password so that I
don’t have to worry about it when I publish my dotfiles.  I simply get prompted
every time I want to send an email.

I use
[mutt](https://nickdesaulniers.github.io/blog/2016/06/18/mutt-gmail-ubuntu/)
to respond to threads when I *don't* have a patch to send.

## Step 2: Make fixes

How do you find a bug to fix?  My general approach to finding bugs in open
source C/C++ code bases has been using static analysis, a different compiler,
and/or more compiler warnings turned on.  The kernel also has an instance of
[bugzilla](https://bugzilla.kernel.org/describecomponents.cgi)
running as an issue tracker.  Work out of a new branch, in case you choose to
abandon it later.  Rebase your branch before submitting (pull early, pull
often).

## Step 3: Thoughtful commit messages

I always run `git log <file I modified>` to see some of the previous commit
messages on the file I modified.

```sh
$ git log arch/x86/Makefile

commit a5859c6d7b6114fc0e52be40f7b0f5451c4aba93
...
    x86/build: convert function graph '-Os' error to warning
commit 3f135e57a4f76d24ae8d8a490314331f0ced40c5
...
    x86/build: Mostly disable '-maccumulate-outgoing-args'
```

The first words of commit messages in Linux are usually
`<subsystem>/<sub-subsystem>: <descriptive comment>`.

Let’s commit, `git commit <files> -s`.  We use the `-s` flag to `git commit` to
add our signoff.  Signing your patches is standard and notes your agreement to
the
[Linux Kernel Certificate of Origin](https://ltsi.linuxfoundation.org/developers/signed-process).

## Step 4: Generate Patch file

`git format-patch HEAD~`.  You can use `git format-patch HEAD~<number of
commits to convert to patches>` to turn multiple commits into patch files.
These patch files will be emailed to the
[Linux Kernel Mailing List (lkml)](https://lkml.org/).
They can be applied with `git am <patchfile>`.  I like to back these files up
in another directory for future reference, and cause I still make a lot of
mistakes with git.

## Step 5: checkpatch

You’re going to want to run the kernel’s linter before submitting.  It will
catch style issues and other potential issues.

```sh
$ ./scripts/checkpatch.pl 0001-x86-build-don-t-add-maccumulate-outgoing-args-w-o-co.patch
total: 0 errors, 0 warnings, 9 lines checked

0001-x86-build-don-t-add-maccumulate-outgoing-args-w-o-co.patch has no obvious style problems and is ready for submission.
```

If you hit issues here, fix up your changes, update your commit with `git
commit --amend <files updated>`, rerun format-patch, then rerun checkpatch
until you’re good to go.

## Step 6: email the patch to yourself

This is good to do when you’re starting off.  While I use
[mutt](https://nickdesaulniers.github.io/blog/2016/06/18/mutt-gmail-ubuntu/)
for responding to email, I use `git send-email` for sending patches.  Once
you’ve gotten a hang of the workflow, this step is optional, more of a sanity
check.

```sh
$ git send-email \
0001-x86-build-require-only-gcc-use-maccumulate-outgoing-.patch
```

You don’t need to use command line arguments to cc yourself, assuming you set
up git correctly, git send-email should add you to the cc line as the author of
the patch.  Send the patch just to yourself and make sure everything looks ok.

## Step 7: get maintainers

Linux is huge, and has a trusted set of maintainers for various subsystems.
The
[MAINTAINERS file](http://elixir.free-electrons.com/linux/latest/source/MAINTAINERS)
keeps track of these, but Linux has a tool to help you figure out where to send
your patch:

```sh
$ ./scripts/get_maintainer.pl 0001-x86-build-don-t-add-maccumulate-outgoing-args-w-o-co.patch
Person A <person@a.com> (maintainer:X86 ARCHITECTURE (32-BIT AND 64-BIT))
Person B <person@b.com> (maintainer:X86 ARCHITECTURE (32-BIT AND 64-BIT))
Person C <person@c.com> (maintainer:X86 ARCHITECTURE (32-BIT AND 64-BIT))
x86@kernel.org (maintainer:X86 ARCHITECTURE (32-BIT AND 64-BIT))
linux-kernel@vger.kernel.org (open list:X86 ARCHITECTURE (32-BIT AND 64-BIT))
```

## Step 8: fire off the patch

```sh
$ git send-email \
--cc person@a.com \
--cc person@b.com \
--cc person@c.com \
--cc x86@kernel.org \
--cc linux-kernel@vger.kernel.org \
0001-x86-build-don-t-add-maccumulate-outgoing-args-w-o-co.patch
```

Make sure to cc yourself when prompted.  Otherwise if you don’t subscribe to
LKML, then it will be difficult to reply to feedback.  It’s also a good idea to
cc any other author that has touched this functionality recently.

## Step 8: monitor feedback

[Patchwork](https://patchwork.kernel.org/project/LKML/list/)
for the LKML is a great tool for tracking the progress of patches.  You should
register an account there.   I highly recommend bookmarking your submitter
link.  In Patchwork, click any submitter, then Filters (hidden in the top
left), change submitter to your name, click apply, then bookmark it.
[Here’s what mine looks like](https://patchwork.kernel.org/project/LKML/list/?submitter=171273).
Not much today, and mostly trivial patches, but hopefully this post won’t age
well in that regard.

Feedback may or may not be swift.  I think my first patch I had to ping a
couple of times, but eventually got a response.

## Step 9: responding to feedback

Update your file, `git commit <changed files> --amend` to update your latest
commit, `git format-patch --subject-prefix="Patch v2" HEAD~`, edit the patch
file to put the changes below the dash below the signed off lines
([example](https://patchwork.kernel.org/patch/9720097/)), rerun checkpatch,
rerun get_maintainer if the files you modified changed since V1.  Next, you
need to find the messageID to respond to the thread properly.

In gmail, when viewing the message I want to respond to, you can click “Show
Original” from the dropdown near the reply button.  From there, copy the
MessageID from the top (everything in the angle brackets, but not the brackets
themselves).  Finally, we send the patch:

```sh
$ git send-email \
--cc nick.desaulniers@gmail.com \
--cc person@a.com \
--cc person@b.com \
--cc person@c.com \
--cc person@d.com \
--cc x86@kernel.org \
--cc linux-kernel@vger.kernel.org \
--in-reply-to 2017datesandletters@somehostname \
0001-x86-build-don-t-add-maccumulate-outgoing-args-w-o-co.patch
```

We make sure to add anyone who may have commented on the patch from the mailing
list to keep them in the loop.  Rinse and repeat 2 through 9 as desired until
patch is signed off/acked or rejected.

Finding out when your patch gets merged is a little tricky; each subsystem
maintainer seems to do things differently.  My first patch, I didn’t know it
went in until a bot at Google notified me.  The maintainers for the second and
third patches had bots notify me that it got merged into their trees, but when
they send Linus a PR and when that gets merged isn’t immediately obvious.

It’s not like Github where everyone involved gets an email that a PR got merged
and the UI changes.  While there’s pros and cons to having this fairly
decentralized process, and while it is kind of is git’s original designed-for
use case, I’d be remiss not to mention that I really miss Github.  Getting your
first patch acknowledged and even merged is
[intoxicating and makes you want to contribute more](https://github.com/nickdesaulniers/What-Open-Source-Means-To-Me#what-open-source-means-to-me);
radio silence has the opposite effect.

Happy hacking!
