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
Select OS for the task is, as preannounced, **CentOS**. This is my standard
choice where no special features are needed (think multimedia packages, for the
most part). The reason of this choice is not fully clear even to me, but I
think my preference goes to it for its small footprint, the availability of
third-party repositories with fairly updated, yet robust, packages that makes
it in general quite stable, even when upgrading. On this subject, I remember
painful hours spent on distro/packages broken upgrade paths with Debian and
derivatives; I'm sure things got better in the last few years, but for the
moment I see little to no reason to move to another OS as my standard choice.

The puppetmaster will run on a VM hosted on a KVM hypervisor; I can hear you
say: _"Why not cloud?"_. I can say there's a reason, which is associated 
mainly to costs, but I'll leave that discussion for another post.

To install VMs on KVM/libvirt I usually rely on **virt-install**, a Python tool
that takes care of generating the relevant XML definition for the VM and booting
it with an installation media that it's removed when installation finishes.  
On CentOS, it can be installed on the hypervisor simply with:
``` bash
yum install python-virtinst
```

I won't go through here on what you need to have a working KVM setup: the only
thing I need to point out for the moment is that I have configured a pool in
libvirt associated to a **LVM Volume Group** called `vmstorage`.

That said, let's start the installation process. We'll create a VM with **1GB
RAM**, **1 virtual CPU** and **10GB of storage**. You can download the CentOS
installation CD .iso from
[here](http://mi.mirror.garr.it/mirrors/CentOS/6.3/isos/x86_64/CentOS-6.3-x86_64-minimal.iso).
I suggest downloading the minimal installation pack since it requires much less
space than the full DVD and provides everything needed to install a bare
minimum system.
Once you have saved the installation media, say, into `/media`, we can proceed
with booting the VM:
```bash
virt-install --name=bernstein --ram=1024 --vcpus=1 \
--cdrom=/media/CentOS-6.3-x86_64-minimal.iso \
--os-type=linux --os-variant=rhel6 \
--disk pool=vmstorage,size=10 --network bridge=br0,model=e1000 \
--video=vga --vnc --connect qemu:///system
```
At this point we don't have any textual access to the VM; however, we can
setup a SSH tunnel to the hypervisor binding the port where the VNC server
for the instance is listening, which can be found with:
``` bash
virsh vncdisplay bernstein
:3
```
After that, if you are working on a Mac, i suggest you to try the excellent
[Chicken](http://sourceforge.net/projects/chicken/) VNC client (I've
tried many, even the Screen Sharing app available by default, but found this is
the app that works best). The latest version (**2.2b2** as time of writing)
can also automatically setup a SSH tunnel for you.

After we have a connection to the VM's display, we can start the installation
process. To fully document it, I'd need to share a screenshot for every step.
Since I'm not a Martian, you will excuse me if I go through this phase by
simply pointing out the relevant configuration options.
Here are the values of the fields filled during installation:

* **Language**: English
* **Keyboard layout**: U.S. English
* **Device Type**: Basic Storage Devices
* **Hostname**: bernstein.derecom.it
* **Timezone**: Europe/Rome
* **Root password**: Hahaha, really you thought I'd have told you?
* **Disk partitioning**: Select _"Replace Existing Linux Systems"_, then tick
  _Review and modify partitioning layout_. Modify default layout as follows:
  * /dev/vda1   500   ext4    /boot
  * /dev/vda2   9739  LVM     os
    * root      7720  ext4    /root
    * swap      2016  swap    -

After the installation process finishes, the VM will be shut down. If we want
to restart it and also have it automatically restarted when server reboots, we
can issue the following commands:
``` bash
virsh autostart bernstein
virsh start bernstein
```
After that, we still need to connect via VNC before we'll be able to SSH into
the machine. Here are the basic steps needed to setup networking (I do not use
DHCP on the main server's subnet):
``` bash
vi /etc/sysconfig/network-scripts/ifcfg-eth0
```
```
DEVICE="eth0"
BOOTPROTO="static"
HWADDR="52:54:00:77:21:AE"
NM_CONTROLLED="no"
ONBOOT="yes"
TYPE="Ethernet"
UUID="8af5d6eb-e0c3-4401-9fd9-cad42c011d3b"
IPADDR=172.16.32.253
NETMASK=255.255.255.0
GATEWAY=172.16.32.254
DNS1=8.8.8.8
```
``` bash
service network restart
```
Also, I setup some mnemonic entries on our authoritative DNS. They will come in
handy very soon:
```
bernstein               A       172.16.32.253
puppet                  CNAME   bernstein
puppetmaster            CNAME   bernstein
```

At this point, we can start do some basic configuration before installing
Puppet.

## Basic Configuration
First thing first: disable SELinux. I haven't found the time yet to study and
understand how SELinux works, so I still prefer to let it out of the game:
``` bash
sed -i -e 's/SELINUX=enforcing/SELINUX=permissive/g' /etc/sysconfig/selinux
setenforce permissive # This way we avoid to reboot
```

Then, I usually install two additional repositories before doing anything else:
[EPEL](http://fedoraproject.org/wiki/EPEL) and 
[RepoForge (was RPMForge)](http://repoforge.org/):
``` bash
rpm -Uvh http://ftp.upjs.sk/pub/mirrors/epel/6/x86_64/epel-release-6-8.noarch.rpm
rpm -Uvh http://pkgs.repoforge.org/rpmforge-release/rpmforge-release-0.5.2-2.el6.rf.x86_64.rpm
```
Then it's good to update available software:
``` bash
yum update
```
The next step regards NTP and shouldn't be needed, but in my case I found it necessary.  
In fact, in virtualized environments clock sync should be provided directly by
hypervisor by exposing an already NTP synced RTC. This was actually the case
when I used to use Xen, and it still works like that in KVM; the only problem
is that the clock still presents jiffies. So, the best thing to do is install
NTP and forget about that:
``` bash
yum install ntp
chkconfig ntpd on
```
Before starting the service, we force an initial synchronization with:
``` bash
ntpd -q
```
Then:
``` bash
service ntpd start
```
Till now, I never encountered problems with the provided defaults. Maybe in the future
I'll also dig into this aspect and customize the configuration; for the moment
I'll leave as it is.

This ends the list of basic configuration tasks I usually do. We can now
stop to shave the yak and proceed to dig into the actual subject of the post.

## Installing Puppet
The awesome guys at [Puppet Labs](http://puppetlabs.com) provide the right
repositories for the job; we can just add them and perform a one-command Puppet
installation:
``` bash
rpm -Uvh http://yum.puppetlabs.com/el/6/products/x86_64/puppetlabs-release-6-6.noarch.rpm
```
If you want to dig more into the topic, here's a 
[very good section](http://docs.puppetlabs.com/guides/puppetlabs_package_repositories.html)
on the Puppet Labs documentation site.

After adding the repo, Puppet server can be installed with:
``` bash
yum install puppet-server
```
This will also take care of installing Ruby. The latest version available in
CentOS is still a 1.8.7 as time of writing, but it's perfectly suitable to run
the Puppet's applications.

## Configuring the Master
### Service Setup
It's no harder than:
``` bash
chkconfig puppetmaster on
service puppetmaster start
```
