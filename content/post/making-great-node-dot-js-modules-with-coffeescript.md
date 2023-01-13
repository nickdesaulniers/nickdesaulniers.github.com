+++
title = "Making Great Node Dot Js Modules With Coffeescript"
date = "2013-08-28"
slug = "2013/08/28/making-great-node-dot-js-modules-with-coffeescript"
Categories = []
+++
[Node.js](http://nodejs.org/)
is a great runtime for writing applications in JavaScript, the language
I primarily develop in.
[CoffeeScript](http://coffeescript.org/)
is a programming language that compiles
to JavaScript.  Why would we write a reusable piece of code, a
[module](http://nodejs.org/api/modules.html)
, in
CoffeeScript?  CoffeeScript is a very high level language and
[beautifully brings together](http://railscasts.com/episodes/267-coffeescript-basics)
my favorite aspects of JavaScript,
Ruby, and Python.  In this tutorial, I'll show you how I create reusable open
source modules for Node.js from CoffeeScript, which is something I recently
discovered while creating a
[playlist parser module](https://github.com/nickdesaulniers/javascript-playlist-parser).
The point is to focus on how to turn a quick hack into a nicely laid
out Node.js module.

The steps are as follows:

1. Turn an idea into a git repo.
2. Add directory structure.
3. Split library functions from testing.
4. Add a Build script.
5. Create node module.
6. Add LICENSE and README.
7. Publish.

First thing's first, we have to have an idea.  It doesn't have to be
revolutionary, just do one thing and do it well.  That is the first rule of
[UNIX ~~fightclub~~ philosophy](http://www.faqs.org/docs/artu/ch01s06.html),
which resonates well within
[the Node.js community](http://blog.izs.me/post/48281998870/unix-philosophy-and-node-js).
When I'm hacking on
something, I start out with a single file to test something out.  Then I
progressively refine the example until it's something reusable.  That way, I
can reuse it, others can reuse it, others can learn from it, and the world can
be a better place.

For this tutorial, I'll show you my process for creating a binding for
[nanomsg](http://nanomsg.org/index.html),
the latest scalability protocol library from the creator of
[ZeroMQ](http://zeromq.org/),
[Martin Sústrik](http://250bpm.com/).
I had played with ZeroMQ in the past and thought that it was really
awesome, and I was excited to see a new library from it's creator, based on C,
since I also really enjoyed his post on why he
[shouldn't have written it in C++](http://250bpm.com/blog:4).

So messing around real quick, let's make sure we have node up to date.  I like
to use
[nvm](https://github.com/creationix/nvm)
and the latest stable minor version of node (stable versions have
even minor patch numbers where versions are in the format `major.minor.patch`,
so v0.11.0 is unstable).  `node -v` -> v0.10.17

Then I need to download and install the library that I'll be dynamically
linking to, build, and install it.

```text
curl -O http://download.nanomsg.org/nanomsg-0.1-alpha.zip && \
unzip nanomsg-0.1-alpha.zip && \
cd nanomsg-0.1-alpha && \
mkdir build && \
cd build && \
../configure && \
make && \
make install
```

We'll use
[node's FFI module](https://github.com/rbranson/node-ffi)
to interface with the dynamically linked library,
because it's easier to write bindings than using
[native addons](http://nodejs.org/api/addons.html),
and
[v8's API has recently changed causing some headaches for native extensions](https://github.com/rvagg/node-addon-examples/blob/master/README.md#compatibility-notes).

`npm install ffi`

We'll be writing the example in CoffeeScript.

`npm install -g coffee-script`

Now to mess around we can create main.coffee based on
[the C++ binding's example](https://github.com/250bpm/cppnanomsg/blob/9becc3d5116ab33a7d2c5f06d68a8fea1b781194/binding.cpp#L29):
```coffeescript main.coffee
ffi = require 'ffi'
assert = require 'assert'

AF_SP = 1
NN_PAIR = 16

nanomsg = ffi.Library 'libnanomsg',
  nn_socket: [ 'int', [ 'int', 'int' ]]
  nn_bind: [ 'int', [ 'int', 'string' ]]
  nn_connect: [ 'int', ['int', 'string' ]]
  nn_send: [ 'int', ['int', 'pointer', 'int', 'int']]
  nn_recv: [ 'int', ['int', 'pointer', 'int', 'int']]
  nn_errno: [ 'int', []]

# test
s1 = nanomsg.nn_socket AF_SP, NN_PAIR
assert s1 >= 0, 's1: ' + nanomsg.nn_errno()

ret = nanomsg.nn_bind s1, 'inproc://a'
assert ret > 0, 'bind'

s2 = nanomsg.nn_socket AF_SP, NN_PAIR
assert s2 >= 0, 's2: ' + nanomsg.nn_errno()

ret = nanomsg.nn_connect s2, 'inproc://a'
assert ret > 0, 'connect'

msg = new Buffer 'hello'
ret = nanomsg.nn_send s2, msg, msg.length, 0
assert ret > 0, 'send'

recv = new Buffer msg.length
ret = nanomsg.nn_recv s1, recv, recv.length, 0
assert ret > 0, 'recv'

console.log recv.toString()
assert msg.toString() is recv.toString(), 'received message did not match sent'
```
`coffee main.coffee` -> hello

This quick example shows that we have something working.  Currently our working
directory should look like:
```text
tree -L 2
.
├── main.coffee
└── node_modules
    └── ffi

2 directories, 1 file
```

## Turn an idea into a git repo

Next up is to create a repository using a version control system like
[git](http://git-scm.com/) and
start saving our work.
[Check in early, check in often](http://www.codinghorror.com/blog/2008/08/check-in-early-check-in-often.html).

Let's add a .gitignore so that were not adding files that really don't need to
be committed.  The node_modules folder is unnecessary because when this node
module is installed, its dependencies will be recursively installed, so
there's no need to commit them to source control.  The swap files are because I
use
[vim](http://www.vim.org/)
and I accidentally commit the swap files from open buffers all the time
like a noob.

```text .gitignore
node_modules/
*.swp
```

Let's turn this into a git repo:
```text
git init && \
git add . && \
git commit -am “initial commit”
```

Up on github, let's [create an uninitialized repo](https://github.com/new)
and push to it:
```text
git remote add origin git@github.com:nickdesaulniers/node-nanomsg.git && \
git push -u origin master
```

So we
[have](https://github.com/nickdesaulniers/node-nanomsg/commit/19211e7520de9384a0d5b0ce4c08a623c4f2e0b9):
```text
tree -L 2 -a
.
├── .gitignore
├── main.coffee
└── node_modules
    └── ffi

2 directories, 2 files
```

## Add directory structure

Now that we have our repo under version control, let's start adding some
structure. Let's create
`src/`, `lib/`, and `test/` directories.  Our CoffeeScript will live in
`src/`, compiled JavaScript will be in `lib/`, and our test code will be in
`test/`.

`mkdir src lib test`

## Split library functions from testing

Now let's move a copy of `main.coffee` into `src/` and one into `test/`.  We
are going to split the library definition away from the testing logic.
```text
cp main.coffee test/test.coffee && \
git add test/test.coffee && \
git mv main.coffee src/nanomsg.coffee
```

This way `git status` tells us:
```text
# On branch master
# Changes to be committed:
#   (use "git reset HEAD <file>..." to unstage)
#
# renamed:    main.coffee -> src/nanomsg.coffee
# new file:   test/test.coffee
#
```

Let's edit src/main.coffee to look like:
```coffeescript src/main.coffee
ffi = require 'ffi'

exports = module.exports = ffi.Library 'libnanomsg',
  nn_socket: [ 'int', [ 'int', 'int' ]]
  nn_bind: [ 'int', [ 'int', 'string' ]]
  nn_connect: [ 'int', ['int', 'string' ]]
  nn_send: [ 'int', ['int', 'pointer', 'int', 'int']]
  nn_recv: [ 'int', ['int', 'pointer', 'int', 'int']]
  nn_errno: [ 'int', []]

exports.AF_SP = 1
exports.NN_PAIR = 16
```

and edit the tests to:
```coffeescript test/test.coffee
assert = require 'assert'
nanomsg = require '../lib/nanomsg.js'

{ AF_SP, NN_PAIR } = nanomsg

s1 = nanomsg.nn_socket AF_SP, NN_PAIR
assert s1 >= 0, 's1: ' + nanomsg.nn_errno()

ret = nanomsg.nn_bind s1, 'inproc://a'
assert ret > 0, 'bind'

s2 = nanomsg.nn_socket AF_SP, NN_PAIR
assert s2 >= 0, 's2: ' + nanomsg.nn_errno()

ret = nanomsg.nn_connect s2, 'inproc://a'
assert ret > 0, 'connect'

msg = new Buffer 'hello'
ret = nanomsg.nn_send s2, msg, msg.length, 0
assert ret > 0, 'send'

recv = new Buffer msg.length
ret = nanomsg.nn_recv s1, recv, recv.length, 0
assert ret > 0, 'recv'

assert msg.toString() is recv.toString(), 'received message did not match sent'
```

Notice how in the test we're including the compiled javascript from `lib/`
which doesn't exist yet?  If you try running `coffee test/test.coffee` it
should crash.  Let's make the compiled version.
`coffee -o lib -c src/nanomsg.coffee`

Once the compiled lib exists, we can run our tests with
`coffee test/test.coffee` and shouldn't see any errors.

Now we should have a little more order, let's
[commit](https://github.com/nickdesaulniers/node-nanomsg/commit/3e3e3918971e2eddbe95e91e0c3cf32e7f8becba).
Hold off on adding
`lib/` to version control, I'll explain why in a bit.
```text
tree -L 2 -C -a -I '.git'
.
├── .gitignore
├── lib
│   └── nanomsg.js
├── node_modules
│   └── ffi
├── src
│   └── nanomsg.coffee
└── test
    └── test.coffee

5 directories, 4 files
```

At this point, if we add features and want to rerun our tests, we need to
execute:

`coffee -o lib -c src/nanomsg.coffee && coffee test/test.coffee`

While this command is simple now and easy to reverse search, anyone else
contributing to you project is going to have to know the commands to run the
tests.  Let's use
[Grunt](http://gruntjs.com/),
the JavaScript task runner, to automate our build and test process.

## Add a Build script

```text
npm install -g grunt-cli && \
npm install grunt-contrib-coffee
```

Create a simple Gruntfile which can also be written in CoffeeScript:
```coffeescript Gruntfile.coffee
module.exports = (grunt) ->
  grunt.initConfig
    coffee:
      compile:
        files:
          'lib/nanomsg.js': ['src/*.coffee']
  grunt.loadNpmTasks 'grunt-contrib-coffee'
  grunt.registerTask 'default', ['coffee']
```

Running `grunt` builds our lib which is a start, so let's
[commit](https://github.com/nickdesaulniers/node-nanomsg/commit/293e7378225c761ec496d9dcd09e1f2d331628a2)
that.


But `grunt` is not running our tests.  And our tests don't have nice output.
Let's change that:
```text
npm install -g mocha && \
npm install chai grunt-mocha-test
```

edit test/test.coffee to:

```coffeescript test/test.coffee
assert = require 'assert'
should = require('chai').should()
nanomsg = require '../lib/nanomsg.js'

describe 'nanomsg', ->
  it 'should at least work', ->
    { AF_SP, NN_PAIR } = nanomsg

    s1 = nanomsg.nn_socket AF_SP, NN_PAIR
    s1.should.be.at.least 0

    ret = nanomsg.nn_bind s1, 'inproc://a'
    ret.should.be.above 0

    s2 = nanomsg.nn_socket AF_SP, NN_PAIR
    s2.should.be.at.least 0

    ret = nanomsg.nn_connect s2, 'inproc://a'
    ret.should.be.above 0

    msg = new Buffer 'hello'
    ret = nanomsg.nn_send s2, msg, msg.length, 0
    ret.should.be.above 0

    recv = new Buffer msg.length
    ret = nanomsg.nn_recv s1, recv, recv.length, 0
    ret.should.be.above 0

    msg.toString().should.equal recv.toString()
```

and modify your gruntfile to add a testing step:

```coffeescript Gruntfile.coffee
module.exports = (grunt) ->
  grunt.initConfig
    coffee:
      compile:
        files:
          'lib/nanomsg.js': ['src/*.coffee']
    mochaTest:
      options:
        reporter: 'nyan'
      src: ['test/test.coffee']

  grunt.loadNpmTasks 'grunt-contrib-coffee'
  grunt.loadNpmTasks 'grunt-mocha-test'

  grunt.registerTask 'default', ['coffee', 'mochaTest']
```

Now when we run `grunt`, our build process will run, then our test process,
then we should see one incredibly happy
[nyan cat](http://www.nyan.cat/).
The
[nyan cat mocha test reporter](http://visionmedia.github.io/mocha/#reporters)
is basically the pinnacle of human intellectual achievement.

```text
grunt
Running "coffee:compile" (coffee) task
File lib/nanomsg.js created.

Running "mochaTest:src" (mochaTest) task
 1   -__,------,
 0   -__|  /\_/\
 0   -_~|_( ^ .^)
     -_ ""  ""

  1 passing (5 ms)


Done, without errors.
```
[Commit time](https://github.com/nickdesaulniers/node-nanomsg/commit/a43bcb3f69ca20bb1902472ecc954317e5fe0fe3).

```text
tree -L 2 -C -a -I '.git'
.
├── .gitignore
├── Gruntfile.coffee
├── lib
│   └── nanomsg.js
├── node_modules
│   ├── ffi
│   ├── grunt
│   └── grunt-contrib-coffee
├── src
│   └── nanomsg.coffee
└── test
    └── test.coffee

7 directories, 5 files
```

## Create node module

Now that we have a more modular design with build and test logic built in,
let's make this module redistributable.  First, let's talk about ignoring
files.  Create a `.npmignore` file that will specify what not to include in the
module that is downloaded.  Node Package Manager,
[npm](https://npmjs.org/),
will
[ignore a bunch of files by default](https://npmjs.org/doc/developers.html#Keeping-files-out-of-your-package)
for us.

```text .npmignore
Gruntfile.coffee
src/
test/
```

Here we're ignoring the `src/` dir, where in our `.gitignore` we are going to
ignore `lib/`.

```text .gitignore
node_modules/
lib/
*.swp
```

Why are we doing this?  Admittedly, none of this is strictly necessary, but
here's why I think it is useful.  When someone is checking out the source, they
don't need the results of the compilation step, as they can make modifications
and would need to recompile anyways.  Adding `lib/nanomsg.js` would just be
another thing to download (though its size is relatively insignificant).
Likewise, when someone downloads the module, they most likely just want the
results of the compilation step, not the source, build script, or test suite.
If I was planned on making the compiled JavaScript accessible to a web browser,
I would not add `lib/` to `.gitignore`, that way it could be referenced from the
github raw URL.
Again, these are generalizations that are not always true.  To make up for not
having the entire source when installed as a module, we'll make up for it by
adding a link to the repo from of manifest, but first let's
[commit](https://github.com/nickdesaulniers/node-nanomsg/commit/61458d964eaaee2ae6501bfbd186a1fe0d03d827)!

Time to create a manifest file that has some basic info about our app.  It's a
pretty good idea to run `npm search <packagename>` before hand to make sure
your planned package name is not taken.  Since we have all of our dependencies
in a row, let's run
`npm init`.

```text
This utility will walk you through creating a package.json file.
It only covers the most common items, and tries to guess sane defaults.

See `npm help json` for definitive documentation on these fields
and exactly what they do.

Use `npm install <pkg> --save` afterwards to install a package and
save it as a dependency in the package.json file.

Press ^C at any time to quit.
name: (nanomsg)
version: (0.0.0)
description = nanomsg bindings
entry point: (index.js) lib/nanomsg.js
test command: grunt
git repository: (git://github.com/nickdesaulniers/node-nanomsg.git)
keywords = ["nanomsg"]
license: (BSD-2-Clause) Beerware
About to write to /Users/Nicholas/code/c/nanomsg/package.json:

{
  "name": "nanomsg",
  "version": "0.0.0",
  "description": "nanomsg bindings",
  "main": "lib/nanomsg.js",
  "directories": {
    "test": "test"
  },
  "dependencies": {
    "chai": "~1.7.2",
    "ffi": "~1.2.5",
    "grunt": "~0.4.1",
    "grunt-mocha-test": "~0.6.3",
    "grunt-contrib-coffee": "~0.7.0"
  },
  "devDependencies": {},
  "scripts": {
    "test": "grunt"
  },
  "repository": {
    "type": "git",
    "url": "git://github.com/nickdesaulniers/node-nanomsg.git"
  },
  "keywords": [
    "nanomsg"
  ],
  "author": "Nick Desaulniers",
  "license": "Beerware",
  "bugs": {
    "url": "https://github.com/nickdesaulniers/node-nanomsg/issues"
  }
}


Is this ok? (yes)
```

That should create for us a nice package.json manifest file for npm.

We can now run our tests with the command `npm test` in addition to `grunt`.
Let's hold off on publishing just yet,
[committing](https://github.com/nickdesaulniers/node-nanomsg/commit/6ac3425005c69683df85f75c49e27c9cb634ada6)
instead.

```
tree -L 2 -C -a -I '.git'
.
├── .gitignore
├── .npmignore
├── Gruntfile.coffee
├── lib
│   └── nanomsg.js
├── node_modules
│   ├── .bin
│   ├── chai
│   ├── ffi
│   ├── grunt
│   ├── grunt-contrib-coffee
│   └── grunt-mocha-test
├── package.json
├── src
│   └── nanomsg.coffee
└── test
    └── test.coffee

10 directories, 7 files
```

## Add LICENSE and README

So we have a module that's almost ready to go.  But how will developers know
how to reuse this code?  As much as I like to
*[view the source, Luke](http://bartaz.github.io/impress.js/#/source)*,
npm will complain without a readme.  The
[readme](https://github.com/nickdesaulniers/node-nanomsg/commit/95e5a7203740f8ab31758f54491d095567accf70)
also looks nice on the github repo.

    # Node-NanoMSG
    Node.js binding for [nanomsg](http://nanomsg.org/index.html).

    ## Usage

    `npm install nanomsg`

    ```javascript
    var nanomsg = require('nanomsg');
    var assert = require('assert');
    var AF_SP = nanomsg.AF_SP;
    var NN_PAIR = nanomsg.NN_PAIR;
    var msg = new Buffer('hello');
    var recv = new Buffer(msg.length);
    var s1, s2, ret;

    s1 = nanomsg.nn_socket(AF_SP, NN_PAIR);
    assert(s1 >= 0, 's1: ' + nanomsg.errno());

    ret = nanomsg.nn_bind(s1, 'inproc://a');
    assert(ret > 0, 'bind');

    s2 = nanomsg.nn_socket(AF_SP, NN_PAIR);
    assert(s2 >= 0, 's2: ' + nanomsg.errno());

    ret = nanomsg.nn_connect(s2, 'inproc://a');
    assert(ret > 0, 'connect');

    ret = nanomsg.nn_send(s2, msg, msg.length, 0);
    assert(ret > 0, 'send');

    ret = nanomsg.recv(s1, recv, recv.length, 0);
    assert(ret > 0, 'recv');

    assert(msg.toString() === recv.toString(), "didn't receive sent message");
    console.log(recv.toString());

Before we publish, let's create a
[license](https://github.com/nickdesaulniers/node-nanomsg/commit/48a55d0c51099f6b90cae5f190a8eb2b94140eae)
file, because though we are making
our code publicly viewable,
[public source code without an explicit license is still under copyright and cannot be reused](https://help.github.com/articles/open-source-licensing#what-happens-if-i-dont-choose-a-license).

```text LICENSE
/*
 * ----------------------------------------------------------------------------
 * "THE BEER-WARE LICENSE" (Revision 42):
 * <nick@mozilla.com> wrote this file. As long as you retain this notice you
 * can do whatever you want with this stuff. If we meet some day, and you think
 * this stuff is worth it, you can buy me a beer in return. Nick Desaulniers
 * ----------------------------------------------------------------------------
 */
```

If you want to be more serious, maybe instead shoot for an MIT or BSD style
license if you don't care what your repo gets used for or GPL style if you do.
[TLDRLegal](http://www.tldrlegal.com/) has a great breakdown on common licenses.

```text
tree -L 2 -C -a -I '.git'
.
├── .gitignore
├── .npmignore
├── Gruntfile.coffee
├── LICENSE
├── README.md
├── lib
│   └── nanomsg.js
├── node_modules
│   ├── .bin
│   ├── chai
│   ├── ffi
│   ├── grunt
│   ├── grunt-contrib-coffee
│   └── grunt-mocha-test
├── package.json
├── src
│   └── nanomsg.coffee
└── test
    └── test.coffee

10 directories, 9 files
```

## Publish

`npm publish`
```text
npm http PUT https://registry.npmjs.org/nanomsg
npm http 201 https://registry.npmjs.org/nanomsg
npm http GET https://registry.npmjs.org/nanomsg
npm http 200 https://registry.npmjs.org/nanomsg
npm http PUT https://registry.npmjs.org/nanomsg/-/nanomsg-0.0.0.tgz/-rev/1-20f1ec5ca2eed51e840feff22479bb5d
npm http 201 https://registry.npmjs.org/nanomsg/-/nanomsg-0.0.0.tgz/-rev/1-20f1ec5ca2eed51e840feff22479bb5d
npm http PUT https://registry.npmjs.org/nanomsg/0.0.0/-tag/latest
npm http 201 https://registry.npmjs.org/nanomsg/0.0.0/-tag/latest
+ nanomsg@0.0.0
```

Finally as a sanity check, I like to make a new folder elsewhere, and run
through the steps in the readme manually to make sure the package is reuseable.
Which is good, since in the readme I
[accidentally forgot](https://github.com/nickdesaulniers/node-nanomsg/commit/c145d3b3a8e85354f1bfb61d8342531ec6bbaa0a)
the `nn_` prefix in
front of errno and recv!

After updating the example in the readme, let's
[bump the version number](https://github.com/nickdesaulniers/node-nanomsg/commit/a6280ca85c4b9cbaac36f5b427bc052961d7e972)
and
republish.  Use `npm version` without arguments to find the current version,
then `npm version patch` to bump it.  You have to commit the readme changes
before bumping the version.  Finally don't forget to rerun `npm publish`.

Our
[final directory structure](https://github.com/nickdesaulniers/node-nanomsg/tree/a6280ca85c4b9cbaac36f5b427bc052961d7e972)
ends up looking like:
```text
tree -L 2 -C -a -I '.git'
.
├── .gitignore
├── .npmignore
├── Gruntfile.coffee
├── LICENSE
├── README.md
├── lib
│   └── nanomsg.js
├── node_modules
│   ├── .bin
│   ├── chai
│   ├── ffi
│   ├── grunt
│   ├── grunt-contrib-coffee
│   └── grunt-mocha-test
├── package.json
├── src
│   └── nanomsg.coffee
└── test
    └── test.coffee

10 directories, 9 files
```
Lastly, I'll
[reach out to](https://github.com/250bpm/nanomsg/pull/122)
Martin Sústrik and let him know that nanomsg has a new binding.

The bindings are far from complete, the test coverage could be better, and the
API is very C like and could use some OO syntactic sugar, but we're at a great
starting point and ready to rock and roll.  If you'd like to help out, fork
[https://github.com/nickdesaulniers/node-nanomsg.git](https://github.com/nickdesaulniers/node-nanomsg.git).

What are some of your thoughts on build steps, testing, and directory layout of
node module?  This
tutorial was definitely not meant to be an authoritarian guide.  I look forward
to your comments on your experiences!
