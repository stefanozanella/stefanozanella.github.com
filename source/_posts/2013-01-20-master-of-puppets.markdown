---
layout: post
title: "Master of Puppets"
date: 2013-01-20 01:16
comments: true
categories: [devops, puppet, howto]
---

Let me start the real first post of this blog (apart from the introduction)
with a citation. Despite the title, I won't talk about music. Instead, I'll
show you a photo:

<!-- More -->
{% img http://farm9.staticflickr.com/8330/8395925627_7fd19c4aba_z.jpg %}

It's a shot I've taken in **Bern** on August 2011; it's the window of a toys 
shop you can find under the cloisters of Gerechtigkeitstraße. Its name? *Antics
und Puppenklinik*, which even without translation projects us directly to the 
focus of this and probably subsequent writings.

As promised in my [first post](http://blog.dontwakethecat.net/blog/2013/01/19/pre-flight-debrief/)
I'll document the setup of a node that will act as the **puppetmaster** for my
company's network. If you don't know what I'm talking about, you probably
landed on the wrong page.  
If, however, you're a sysadmin or an operations guy
and still don't know what I'm talking about, you should **IMMEDIATELY** go
checking out [Puppet Documentation](http://docs.puppetlabs.com/) on the 
[Puppet Labs website](http://puppetlabs.com/).

Before starting, here's a brief overview. We'll start by installing a fresh
**CentOS 6.3** on a **KVM** virtual machine, then we will step into installing
**Puppet** and configuring it so it can start talking with other machines. For
sure, it isn't rocket science; but again, I'm just trying to document the most
I can about what I do.

## Installing Bare OS

## Installing Puppet
