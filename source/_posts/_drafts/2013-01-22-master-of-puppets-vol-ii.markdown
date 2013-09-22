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
NO --> [Publish over SSH](https://wiki.jenkins-ci.org/display/JENKINS/Publish+Over+SSH+Plugin)
NO --> [SSH Agent Plugin](https://wiki.jenkins-ci.org/display/JENKINS/SSH+Agent+Plugin)

```
yum install git
usermod -s /bin/bash puppet
mkdir ~puppet/.ssh
vi ~puppet/.ssh/authorized_keys
...show options
chown puppet:puppet -R ~puppet/.ssh/
chmod og-rw -R ~puppet/.ssh/
```

Configure job in Jenkins
--> Setup git URL

Configure Gerrit trigger

Configure Gerrit to handle non-interactive user
* Perform initial connection with Jenkins user
* Add SSH public key
* Login as Administrator
* Groups -> List -> Non-interactive users
* Add user Jenkins to group

Configure access rights for non-interactive users
* Login as administrator
* Projects -> All-Projects -> Access
* Reference: refs/* -> Label Verified +1, -1 Non-interactive users; Submit
  ALLOW Registerd Users
Configure plugin in Jenkins

Setup connection params
Setup default submit when +2 --> add '--submit' to Successfull verify command
<Add screenshot>

Setup global username and email for Git Plugin in Manage Jenkins






Create Gerrit project
ssh -p 29418 stefano.zanella@review.derecom.it gerrit create-project --name puppetmaster-config
git clone ssh://stefano.zanella@review.derecom.it:29418/puppetmaster-config.git
cd puppetmaster-config
git config remote.origin.push refs/heads/*:refs/for/*
scp -P 29418 stefano.zanella@review.derecom.it:hooks/commit-msg .git/hooks

Add puppet configuration
scp -r root@puppet.derecom.it:/etc/puppet/* .
touch manifests/.gitkeep
touch modules/.gitkeep
git add -A
git commit -m "Store current configuration"
git push

Create Jenkins job for reviewing configuration changes
puppetmaster-config-review
Source Code Management: Git
Repositories: ssh://jenkins@review.derecom.it:29418/puppetmaster-config
Click on Advanced
Refspec $GERRIT_REFSPEC
Branches to build $GERRIT_BRANCH
Click on Advanced
Choosing Strategy = Gerrit Trigger
Build Triggers
Select Gerrit event
Click Advanced
Verify votes:
0, 1, -1, 0
0, 2, -2, 0
Trigger on Patchset Created
Gerrit project: Type Plain, Pattern puppetmaster-config, Branches: Type Path
Branches **
Build -> Add build step -> Execute Shell
Command:
echo "Nothing to check, build OK"
exit 0

Add Gerrit host to known hosts on the slave. Also add Jenkins public/private
key to slave .ssh, otherwise it won't be able to connect to Gerrit.


Add Jenkins job to update configuration on Puppet Master
puppetmaster-config-deploy
Source Code Management: None
Build Triggers: Gerrit Event
Gerrit Trigger:
Silent Mode on
Trigger on: Change merged
Gerrit project: as above
Build: execute shell script on remote host using ssh
SSH Site: puppet@puppet.derecom.it:22
Command: cd /etc/puppet && git pull

Note that this kind of trigger mode cannot be tested in Jenkins with the
Gerrit tool in the dashboard's homepage. You must actually push a change and
test the whole workflow.

Swap configuration directory with repository
cp -r /etc/puppet /etc/puppet.bak
git clone https://review.derecom.it/puppetmaster-config /etc/puppet

Add Jenkins job to update configuration on Puppet Master
puppetmaster-config-deploy
Source Code Management: None
Build Triggers: Gerrit Event
Gerrit Trigger:
Silent Mode on
Trigger on: Change merged
Gerrit project: as above
Build: execute shell script on remote host using ssh
SSH Site: puppet@puppet.derecom.it:22
Command: source .bash_profile && cd /etc/puppet && git pull && rake

Rake default task: DESCRIPTION!!!

Note that this kind of trigger mode cannot be tested in Jenkins with the
Gerrit tool in the dashboard's homepage. You must actually push a change and
test the whole workflow.

Add CA to ca-bundle
cat /etc/pki/tls/certs/Derecom-Auth-CA-bundle.crt | ssh root@puppet.derecom.it "cat >> /etc/pki/tls/certs/ca-bundle.crt"

Install rbenv on puppet master
yum groupinstall 'development tools'
yum install zlib-devel libxml2-devel libxslt-devel openssl-devel
su - puppet -s /bin/bash
git clone git://github.com/sstephenson/rbenv.git ~/.rbenv
git clone git://github.com/sstephenson/ruby-build.git ~/.rbenv/plugins/ruby-build
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bash_profile
echo 'eval "$(rbenv init -)"' >> ~/.bash_profile
