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
The next step is the most important: setting up SSL. It must be noted that
Puppet Master is able to handle a PKI itself, and provides a number of
facilities for the user in this sense (read automatic certificate signing or
command line certificate signing confirmation). I see however (at least) two
caveats with this approach:

* if you enable automatic certificate signing, even with (sub)domain control
  (more info [here](http://docs.puppetlabs.com/guides/configuring.html) under
  the `autosign.conf` section), you are relaxing your security constraints
  since anyone that generates a request with a suitable domain (that can easily
  be spotted) can obtain a valid certificate within the whole infrastructure. 
  I'm not a security paranoid (should do I?), but that's worth noting
* manually signing certificates via command line on the master is, in my
  opinion, not very DevOps style. Or maybe it is, but the truth is that I'm too
  lazy to consider doing that task manually. (Yes, it surely can be automated
  in some way, but at the moment I feel too lazy also for thinking/searching
  about this).

These two considerations are surely arguable and could be easily ignored, 
but there's another aspect to consider: in the infrastructure I'm managing, 
I do an heavy use of SSL certifcates. 
Basically, every host already has a certificate, which is signed
by a central PKI I built (sorry, no time to talk here about this, but I promise
I'll also write about how that works). These certificate are used for a range
of tasks, from classic HTTPS on Web-exposed services, to bidirectional crypted
communication between services (mainly LDAPS and PostgreSQL). 
I generally sign a certificate per host and use it to identify the host for 
each service it provides/consume on the infrastructure.  
That said, I would like to rely on this already working PKI also for managing
communication between puppet services.

This task can be accomplished by first providing a certificate/private key pair
to the puppetmaster, and then tweaking some configuration options in
`puppet.conf`.  
We start by generating and storing on the master the objects needed to setup an
SSL-enciphered communication: the **SSL certificate**, its associated **private key** 
and the **Certificate Authority chain** to be trusted as the origin of all
certificates.
We start by generating a **CSR** on the node that contains the PKI (which is basically a
self-signed certificate that can sign other certificates). The subject of this
object is the node on which the puppet master runs; since this node at the
moment has three names (`puppet`, `puppetmaster` and `bernstein`) we can
configure the CSR to also hold additional DNS names. This is done by providing
a configuration file to the `openssl` CLI tool that contains an extension which
specifies the alternative names for the subject.  
To build this configuration file, we can copy the shipped `openssl.cnf` and
tweak its configuration:
``` bash
cp /etc/pki/tls/openssl.cnf /etc/pki/tls/bernstein.derecom.it.cnf
vi /etc/pki/tls/bernstein.derecom.it.cnf
```
``` ini
...
[ req ]
...
req_extensions = v3_req # The extensions to add to a certificate request
...
[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = puppet.derecom.it
DNS.2 = puppetmaster.derecom.it
...
```
Note that it is **essential** for the correct operation of the master that the
at least one of the DNS names reported correspond to the FQDN the agents use
to connect to the server. Otherwise, they will complain that the server it's
not who is expected to be.

Now that the configuration is in place, we can generate the CSR and the private
key with the following command (output is reported, also):
``` bash
openssl req -new -nodes -subj '/C=IT/ST=Veneto/L=Padova/O=Derecom srl/OU=IT/CN=bernstein.derecom.it' \
-keyout /etc/pki/tls/private/bernstein.derecom.it.key \
-out /etc/pki/tls/bernstein.derecom.it.csr \
-config /etc/pki/tls/bernstein.derecom.it.cnf
Generating a 2048 bit RSA private key
...+++
............................................................................................................................................................................................+++
writing new private key to '/etc/pki/tls/private/bernstein.derecom.it.key'
-----
```

At this point we have a signing request, which can be thought as a certificate
that cannot be trusted by anyone yet, because nobody can assure the certificate
is authentic. For this reason, it must be _signed_ by a trusted authority.  
We do this step using our self-signed CA: basically a certificate that can
"sign" other certificates and that is supposed to be trusted by the network.
Again, I leave out here the details to setup such a self-signed authority; you
need to take for granted that the following command works as expected (output
appended):
``` bash
openssl ca -policy policy_anything -in /etc/pki/tls/bernstein.derecom.it.csr \
-out /etc/pki/tls/bernstein.derecom.it.crt -config /etc/pki/Auth-CA/openssl.cnf
Using configuration from /etc/pki/Auth-CA/openssl.cnf
Enter pass phrase for /etc/pki/Auth-CA/private/Derecom-Auth-CA.key:
Check that the request matches the signature
Signature ok
Certificate Details:
        Serial Number:
            d6:9b:51:24:6d:b2:23:ae
        Validity
            Not Before: Jan 21 16:19:49 2013 GMT
            Not After : Jan 20 16:19:49 2018 GMT
        Subject:
            countryName               = IT
            stateOrProvinceName       = Veneto
            localityName              = Padova
            organizationName          = Derecom srl
            organizationalUnitName    = IT
            commonName                = bernstein.derecom.it
        X509v3 extensions:
            X509v3 Basic Constraints: 
                CA:FALSE
            Netscape Comment: 
                OpenSSL Generated Certificate
            X509v3 Subject Key Identifier: 
                E1:87:A2:E3:17:2A:A5:80:09:DC:2D:B0:60:CF:2F:A8:CC:96:C5:1E
            X509v3 Authority Key Identifier: 
                keyid:00:E7:0E:B6:D5:20:C6:00:9A:91:DF:69:3F:8A:21:4F:01:DD:60:5F

            X509v3 Key Usage: 
                Digital Signature, Non Repudiation, Key Encipherment
            X509v3 Subject Alternative Name: 
                DNS:puppet.derecom.it, DNS:puppetmaster.derecom.it
Certificate is to be certified until Jan 20 16:19:49 2018 GMT (1825 days)
Sign the certificate? [y/n]:y


1 out of 1 certificate requests certified, commit? [y/n]y
Write out database with 1 new entries
Data Base Updated
```

The output of this command is the signed certificate. We can take this file and
the private key created before and copy to the puppetmaster node. Also, we copy
a third object, which is the CA itself (actually the _chain of CA_, since I'm
technically using an Intermidiate Certificate Authority in this case - you can learn
more about the topic
[here](http://en.wikipedia.org/wiki/Intermediate_certificate_authorities)).  
The CA must be installed on every node that makes use of SSL with certificates
that originated in our PKI because otherwise certificates themselves could not
be verified:
``` bash
scp /etc/pki/tls/bernstein.derecom.it.crt puppet.derecom.it:/etc/pki/tls
scp /etc/pki/tls/private/bernstein.derecom.it.key puppet.derecom.it:/etc/pki/tls/private
scp /etc/pki/tls/certs/Derecom-Auth-CA-bundle.crt puppet.derecom.it:/etc/pki/tls
```

Now that the Puppet Master has everything he needs to have, we can actually
update its configuration. Basically, we want to prevent the master to act as a
Certificate Authority: he will only be able to verify its peers using the
available trusted CA. Also, we want the agent to prevent generating a CSR if no
certificate available: it will our responsibility to provide clients with a
correct certificate.  
The configuration is done again in `/etc/puppet/puppet.conf`. The options
stored in the `main` section are valid also for the agent; instead, we disable
the CA in a `master` section:
``` ini
[main]
...
    # Where SSL certificates are kept.
    # The default value is '$confdir/ssl'.
    ssldir = /etc/pki/tls
    privatekeydir = $ssldir/private
    publickeydir = $ssldir
    certdir = $ssldir
    requestdir = $ssldir
    hostcert = $certdir/$certname.crt
    hostpubkey = $certdir/$certname.pub
    hostprivkey = $privatekeydir/$certname.key
    hostcsr = $requestdir/$certname.csr
    cadir = $ssldir
    cacert = $cadir/Derecom-Auth-CA-bundle.crt
    localcacert = $cadir/Derecom-Auth-CA-bundle.crt
    ca_name = Derecom Auth CA

[master]
    ca = false
...
```

With this configuration in place, we still need to do an additional step due to
(what I consider as) a bug in Puppet; in fact, despite the configuration
options that clearly set the template of the SSL certifcate, key and authority
filenames to look for, Puppet still tries to look for files that end with the
`.pem` suffix. Also, it'll look for a file called `cert.pem` as the CA.  
A quick workaround is to setup a couple of symbolic links so to make puppet
master happy:
``` bash
ln -s /etc/pki/tls/bernstein.derecom.it.crt /etc/pki/tls/bernstein.derecom.it.pem
ln -s /etc/pki/tls/private/bernstein.derecom.it.key /etc/pki/tls/private/bernstein.derecom.it.pem
ln -s /etc/pki/tls/Derecom-Auth-CA.bundle /etc/pki/tls/cert/pem
```

Hopefully this will be fixed one day; for the moment we put the workaround in
place and forget about it.

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

Notice that we don't enable the provided service for the master. This is
because we'll setup a proxy with Nginx and Passenger as the following task.
