---
layout: post
title: "My (brief) journey on release management"
date: 2013-02-07 20:10
comments: true
categories: [devops, package, deployment]
published: false
---

Choice of the format: gem or deb?
Deb because:
- I'm managing an application, not a library (cite Yehuda Katz post about this
  subject)
- I want to make the application self contained so to avoid dependency
  conflicts as much as possible (e.g. with puppet)
- I still can't figure out how to do the latter with gems

Choice of the filesystem layout -> Where to put files?
Quote Filesystem Hierarchy Standard:
> /opt is reserved for the installation of add-on application software packages.
> 
> A package to be installed in /opt must locate its static files in a separate
> /opt/<package> or /opt/<provider> directory tree
> 
> Programs to be invoked by users must be located in the directory
> /opt/<package>/bin
> 
> Host-specific configuration files must be installed in /etc/opt
> 
> 
> /usr is the second major section of the filesystem. /usr is shareable,
> read-only data. That means that /usr should be shareable between various
> FHS-compliant hosts and must not be written to. Any information that is
> host-specific or varies with time is stored elsewhere.
> 
> Large software packages must not use a direct subdirectory under the /usr
> hierarchy.
> 
> /usr/lib includes object files, libraries, and internal binaries that are not
> intended to be executed directly by users or shell scripts. [22]
> 
> Applications may use a single subdirectory under /usr/lib. If an application
> uses a subdirectory, all architecture-dependent data exclusively used by the
> application must be placed within that subdirectory.
> 
> /usr/bin : Most user commands
> Purpose
> 
> This is the primary directory of executable commands on the system.
Choice of /usr/lib + symlinks: depict idea, chosen because I feel better about
it, and also seen more often in real packages (like OpenJDK). /opt seems to be
used "manually" when installing pre-backed applications (like custom JVM
versions or Java application server that just need to get extracted into a
folder).

Classic packaging steps (as per common packaging guides):
- download source package tarball
- extract
- ./configure
- makea -> compile, almost not necessary in ruby
- make install -> puts file in right place on filesystem, can be fooled to
  install to a custom dir usually specifying (at least) a --prefix option in configure


Identified steps for building/releasing the package:
- generate source package -> depict structure, make rake task (cite puppetlabs source)
 - tar format, just contains the full source tree (something like a clone of a
   source repository)
