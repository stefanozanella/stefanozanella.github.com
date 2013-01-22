---
layout: post
title: "Master of Puppets (vol. II)"
date: 2013-01-22 15:14
comments: true
categories: [devops, puppet, howto]
published: false
---

I'll continue here what I'll started in the 
[previous post](http://blog.dontwakethecat.net/blog/2013/01/20/master-of-puppets/). 
We now have a working Puppet Master installation; still, if we want to update
the configuration of the master itself or of one of the managed nodes, we need
to log into the virtual machine and manually edit the files in `/etc/puppet`.  
What we want to achieve in the long term, though, is to not need to SSH into
machines for this kind of normal operations tasks.  

<!-- More -->
So we need a way to extract the configuration contained in the `/etc/puppet`
directory so that we can manage it from our laptop with our favorite editor.
While we're at this, we also put that configuration under version control,
which is always a good thing to do.  
The two friends that will help us in this little journey are
[Jenkins](http://jenkins-ci.org/) and the 
[Gerrit Code Review](http://code.google.com/p/gerrit/) application. 

Plugins needed:
[SSH Plugin](https://wiki.jenkins-ci.org/display/JENKINS/SSH+plugin)
[Publish over SSH](https://wiki.jenkins-ci.org/display/JENKINS/Publish+Over+SSH+Plugin)
[SSH Agent Plugin](https://wiki.jenkins-ci.org/display/JENKINS/SSH+Agent+Plugin)
