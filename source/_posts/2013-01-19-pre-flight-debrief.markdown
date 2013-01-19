---
layout: post
title: "Pre-flight debrief"
date: 2013-01-19 20:20
comments: true
categories: devops
---

I'm writing this post as a sort of disclamer for the posts that will follow in
the (hopefully near) future.

I became a fan of documentation near three years ago. Since then, I mantained
(and am still mantaining) a sort of private wiki where I try do document
whatever I do. In some cases, this practice saved my ass: since for the moment
I don't work in a team, I'm forced to continuously move between tasks that span
from the full Dev and Ops range. Sometimes quite some time can pass until I
need to return to a previous task to fix/improve things. This can be daunting,
since memory tends to be pretty unreliable in these cases; having a neat trace
of what I did helped me to quickly restore the archived knowledge.  
Over time, documenting **while** doing has become almost a natural practice for 
me, at least for what concerns operations tasks.

As stated above, till now I kept this documentation almost private, just
visible to interested friends that wanted to dig from time to time into what I
did; now I feel this practice has to change.  
So, I decided to at least try to make my efforts public. I'm sure there won't
be crowds of developers or sysadmins waiting for my next post; I'm not doing it
for this reason. I'll just try to use the lever that is what in my opinion is
the biggest benfit of open source culture: it forces you to be better at what
you do.

So, this little detour over my recent history just to say one basic thing: I'm
gonna store here almost everything. So, expect to find obvious things as well
as not-so-obvious ones; actually, I suspect the former will overweight the latter
at least for the moment :)

Having clarified this basic concept, let's see what the sudden future reserves:
I'm gonna talk a lot in the following months about my efforts to automate the
infrastructure I'm actually managing. This effort started slightly more than a
year ago when I discovered the excellent ["Continuos Delivery" book](http://continuousdelivery.com/) 
by [Jez Humble](https://twitter.com/jezhumble) and David Farley. 
Meanwhile, I've done a lot to try to add some solid
ground on the infrastructure while maintaining the existent. I now reached a
point in which I can start to implement all those nice automation practices.  
I'll try to convert documentation I've already written into something suitable
for the public audience; but, since I don't want to start yak shaving right
from the beginning, I'll try to stay pragmatic enough and start by writing
currently on-air tasks. For this reason, I apologize in advance if some pieces
will miss from what I'll write.

I force myself now to stop adding to this post; enough self-centered writing
for my tastes. Better to start writing something more practical; I'll start
with something very very basic and in very very little DevOps style: installing 
a Puppet master node.
