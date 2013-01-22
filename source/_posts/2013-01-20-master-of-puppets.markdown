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
Puppet's applications.

## Configuring the Master
We want to configure the puppetmaster both as the master as well as an agent
for itself. Both configurations are done in `/etc/puppet/puppet.conf`.  
We start by configuring the most basic options for both the master and the
agent; then we'll move into configuring the right SSL support for our
infrastructure.

### Basic configuration options
For what concerns the master, the only option we touch for the moment is the
one that enables distribution of custom facts and types from the server to the
agents (plugin sync):
``` ini
[main]
...
pluginsync = true
...
```
This same option, since it is set in the `main` section, also apply for the
agent.

Also, for the agent, we set the appropriate name for the master via the
`server` directive. By default the agent would look for an host named `puppet`;
since our master will be reachable via its FQDN, we need to be explicit about
this. This option belongs to the `agent` section:
``` ini
[agent]
...
server = puppet.derecom.it
...
```

### SSL Setup
_I lost almost an afternoon writing about my intended SSL setup, just to
discover that as of current version (3.0.2), Puppet pose serious limits on
swapping its internal PKI management. In my head, I wanted to completely
disable it and provide certificates myself from my already working PKI.  
Unfortunately, there's a serious incompatibility in how OpenSSL creates
certificates subjects and what Puppet intends as a "valid" subject. If you
want, you can read more [here](http://projects.puppetlabs.com/issues/15561).  
Until the issue is resolved, I'm forced to rely on default PKI management; for
this to work, no additional configuration steps are needed._

### Service Setup
Now we're finally ready to start the master. With everything in place, it's no harder than:
``` bash
service puppetmaster start
```

Similarly, for the agent:
``` bash
chkconfig puppet on
service puppet start
```

Notice that we don't enable the provided service for the master at boot. This is
because we'll setup a proxy with Nginx and Passenger as the following task. We
just run it once so the master can create its own PKI, which is needed in order
to accomplish the following section.

### Proxying Master with Nginx + Passenger
As suggested by 
[official documentation](http://docs.puppetlabs.com/guides/installation.html#post-install),
the default WEBrick server is not suitable for real-life workloads. So, here
we'll setup the master to receive requests from a proxy instead of directly
handling them. Selected stack is [Apache](http://httpd.apache.org/) and 
[Phusion Passenger](http://www.modrails.com/). This is because the version of
nginx that can be found into Passenger or EPEL repos doesn't ship with the
`ngx_headers_more` module, needed to set additional headers when processing
client requests.
The procedure depicted here is adapted from the relative
[article](http://docs.puppetlabs.com/guides/passenger.html) in Puppet's
documentation.

We start by adding the Phusion Passenger repository, which provides an updated
version of Passenger itself, as well as some other goodies we don't need ATM:
``` bash
yum install http://passenger.stealthymonkeys.com/rhel/6/passenger-release.noarch.rpm
```

Then we install the relevant applications:
``` bash
yum install httpd mod_passenger mod_ssl
```

At this point we need to setup a root for the Rack application that will serve
Puppet requests. A default Rack application expects three files:

* a `config.ru` which can be called by Rack itself
* a `public/` folder
* a `tmp/` folder

We start with the required folders, setting our root at `/opt/puppetmaster`:
``` bash
mkdir -p /opt/puppetmaster/{tmp,public}
```

Thankfully, Puppet Labs ships a working `config.ru` file into the master's
package:
``` bash
cp /usr/share/puppet/ext/rack/files/config.ru /opt/puppetmaster/
chown puppet:puppet /opt/puppetmaster/config.ru
```

The only thins that's left to do is to setup the required Apache virtual host.
Again, Puppet Labs provides a sample configuration file that can be used as a
base:
``` bash
cp /usr/share/puppet/ext/rack/files/apache2.conf /etc/httpd/conf.d/puppetmaster.conf
```

The content of the file must be edited to reflect actual paths as follows:
``` 
# Tunable settings
PassengerHighPerformance on
PassengerMaxPoolSize 12
PassengerPoolIdleTime 1500
# PassengerMaxRequests 1000
PassengerStatThrottleRate 120
RackAutoDetect Off
RailsAutoDetect Off

Listen 8140

<VirtualHost *:8140>
        SSLEngine on
        SSLProtocol -ALL +SSLv3 +TLSv1
        SSLCipherSuite ALL:!ADH:RC4+RSA:+HIGH:+MEDIUM:-LOW:-SSLv2:-EXP

        SSLCertificateFile
/var/lib/puppet/ssl/certs/bernstein.derecom.it.pem
        SSLCertificateKeyFile
/var/lib/puppet/ssl/private_keys/bernstein.derecom.it.pem
        SSLCertificateChainFile /var/lib/puppet/ssl/certs/ca.pem
        SSLCACertificateFile    /var/lib/puppet/ssl/certs/ca.pem
        SSLCARevocationFile     /var/lib/puppet/ssl/crl.pem
        SSLVerifyClient optional
        SSLVerifyDepth  1
        # The `ExportCertData` option is needed for agent certificate
expiration warnings
        SSLOptions +StdEnvVars +ExportCertData

        # This header needs to be set if using a loadbalancer or proxy
        RequestHeader unset X-Forwarded-For

        RequestHeader set X-SSL-Subject %{SSL_CLIENT_S_DN}e
        RequestHeader set X-Client-DN %{SSL_CLIENT_S_DN}e
        RequestHeader set X-Client-Verify %{SSL_CLIENT_VERIFY}e

        DocumentRoot /opt/puppetmaster/public/
        RackBaseURI /
        <Directory /opt/puppetmaster/>
                Options None
                AllowOverride None
                Order allow,deny
                allow from all
        </Directory>
</VirtualHost>
```

At this point, we just need to enable Apache at boot and replace the running
puppetmaster daemon:
``` bash
chkconfig httpd on
service puppetmaster stop
service httpd start
```

### Firewall setup
Before rolling, we need to setup iptables and open the relevant port in order
for the master to be reachable:
``` bash
vi /etc/sysconfig/iptables
```
```
...
-A INPUT -m state --state NEW -m tcp -p tcp --dport 8140 -j ACCEPT
...
```
``` bash
service iptables restart
```

## Testing the installation
A quick test can be done on the master itself by running a one-shot agent, like
this:
``` bash
puppet agent --test --debug
```

If everything is working as expected, the last non-debug line should report a
notice that catalog run went fine:
```
Notice: Finished catalog run in 0.04 seconds
```

Obviously, the master doesn't need to sign the certificate, since the agent is
picking the same certificate the master created on the first run. For the other
agents, we'll need to manually sign certificates when new CSRs arrive.
