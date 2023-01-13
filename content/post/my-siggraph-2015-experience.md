+++
title = "My Siggraph 2015 Experience"
date = "2015-08-14"
slug = "2015/08/14/my-siggraph-2015-experience"
Categories = []
+++
I was recently lucky enough to get to attend my first SIGGRAPH conference this
year.  While I didn't attend any talks, I did spend some time in the expo. Here
is a collection of some of the neat things I saw at SIGGRAPH 2015.  Sorry it's
not more collected; I didn't have the intention of writing a blog post until
after folks kept asking me "how was it?"

##VR
Most booths had demos on VR headsets.  Many were DK2's and GearVR's.  AMD and
NVIDIA had Crescent Bay's (next gen VR headset).  It was noticeably lighter than
the DK2, and I thought it rendered better quality.  It had nicer cable bundling,
and
headphones built in, that could fold up and lock out of the way that made it
nice to put on/take off.  I also tried a Sony Morpheus.  They had a very
engaging demo that was a tie in to the upcoming movie about tight rope walking,
"The Walk".  They had a thin PVC pipe taped to the floor that you had to
balance on, and a fan, and you were tight rope walking between the Twin Towers.
Looking down and trying to balance was terrifying.  There were some demos with
a strange mobile VR setup where folks had a backpack on that had an open laptop
hanging off the back and could walk around.  Toyota and Ford had demos where you
could inspect their vehicles in virtual space.  I did not see a single HTC/Valve
Vive at SIGGRAPH.

{% img center /images/siggraph/s8.jpg %}
{% img center /images/siggraph/s1.jpg %}
{% img center /images/siggraph/s3.jpg %}
{% img center /images/siggraph/s2.jpg %}
{% img center /images/siggraph/s4.jpg %}

##AR
Epson had some AR glasses. They were very glasses friendly, unlike most VR
headsets.  The nose piece was flexible, and if you flattened it out, the headset
could rest on top of your glasses and worked well.  The headset had some very
thick compound lenses.  There was a front facing camera and they had a simple
demo using image recognition of simple logos (like QR codes) that helped provide
position data.  There were other demos with orientation tracking that worked
well.  They didn't have positional sensor info, but had some hack that tried to
estimate positional velocity off the angular momentum (I spoke with the
programmer who implemented it).  https://moverio.epson.biz/

##Holograms
There was a demo of holograms using tilted pieces of plastic arranged in a box.
Also, there was a multiple (200+) projector array that projected a scene onto a
special screen.  When walking around the screen, the viewing angle always seemed
correct.  It was very convincing, except for the jarring restart of the animated
loop which could be smoothed out (think looping/seamless gifs).

{% img center /images/siggraph/s5.jpg %}

##VR/3D video
Google cardboard had a booth showing off 3D videos from youtube.  I had a hard
time telling if the video were stereoscopic or monoptic since the demo videos
only had things in the distance so it was hard to tell if parallax was
implemented correctly.  A bunch of booths were showing off 3D video, but as far
as I could tell, all of the correctly rendered stereoscopic shots were computer
rendered.  I could not find a single instance with footage shot from a
stereoscopic rig, though I tried.

##Booths/Expo
NVIDIA and Intel had the largest booths, followed by Pixar's Renderman.  Felt
like a GDC event, smaller, but definitely larger than GDC next.  More focus on
shiny photorealism demos, artistic tools, less on game engines themselves.

##Vulcan/OpenGL ES 3.2
Intel had demos of Vulcan and OpenGL ES 3.2.  For 3.2 they were showing off
tessellation shaders, I think.  For the Vulcan demo, they had a cool demo showing
how with
a particle demo scene rendered with OpenGL 4, a single CPU was pegged, it was
using a lot of power, and had pretty abysmal framerate.  When rendering the same
scene with Vulcan, they were able to more evenly distribute the workload across
CPUs, achieve higher framerate, while using less power.  The API to Vulcan is
_still_ not published, so no source code is available. It was explained to me
that Vulcan is still not thread safe; instead you get the freedom to implement
synchronization rather than the driver.

##Planetarium
There was a neat demo of a planetarium projector being repurposed to display
an "on rails" demo of a virtual scene.  You didn't get parallax since it was
being projected on a hemisphere, but it was neat in that like IMAX your entire
FOV was encompassed, but you could move your head, not see any pixels, and
didn't experience any motion sickness or disorientation.

{% img center /images/siggraph/s7.jpg %}

##X3D/X3DOM
I spoke with some folks at the X3D booth about X3DOM.  To me, it seems like a
bunch of previous attempts have kind of added on too much complexity in an
effort to support every use case under the sun, rather than just accept
limitations, so much so that getting started writing hello world became
difficult.  Some of the folks I spoke to at the booth echoed this sentiment, but
also noted the lack of authoring tools as things that hurt adoption.  I have
some neat things I'm working on in this space, based on this and other prior
works, that I plan on showing off at the upcoming BrazilJS.

##Maker Faire
There was a cool maker faire, some things I'll have to order for family members
(young hackers in training) were [Canny bots](http://cannybots.com/),
[eBee](http://ebeeproject.net/) and [Piper](http://www.withpiper.com/).

{% img center /images/siggraph/s6.jpg %}

##Experimental tech
Bunch of neat input devices, one I liked used directional sound as tactile
feedback.  One demo was rearranging icons on a home screen.  Rather than touch
the screen, there was a field of tiny speakers that would blast your finger with
sound when it entered to simulate the feeling of vibration. It would vibrate
to let you know you had "grabbed" and icon, and then drag it.

##Book Signing
This was the first time I got to see my book printed in physical form!  It
looked gorgeous, hardcover printed in color.  I met about half of the fellow
authors who were also at SIGGRAPH, and our editor.  I even got to meet Eric
Haines, who reviewed my chapter before publication!

{% img center /images/siggraph/s9.jpg %}
{% img center /images/siggraph/s10.jpg %}
{% img center /images/siggraph/s11.jpg %}

