---
layout: post
title: "Puppet Module Skeleton, CI Friendly"
date: 2013-02-12 20:24
comments: true
categories: devops, puppet, ruby, jenkins
---

## Introduction
I'll create here a skeleton that can be useful for developing Puppet modules that
adhere to some of the basic good development practices, such as testing and
continuous integration.
Why don't I just use `puppet module generate`, you say? Because I want to have a more
complete testing framework in place, and also have some additional 
facilities for reporting on tests. Though, using Puppet's **module** face isn't
incompatible with what depicted here; you can just run it before performing the
steps that follow, ignoring the creation of module's directories.

<!-- More -->
## Basic structure
We start by creating the directory where the module will live:

```bash
mkdir <modulename>
cd <modulename>
```

After that, we need to setup the basic layout of a standard Puppet module:

```bash
mkdir manifests files lib templates
```

Then, we create the main manifest of the module (this will come in
handy shortly):

```bash
vi manifests/init.pp
```

The content will be pretty basic for the moment. It will contain the same
documentation sketch that `puppet module` generates, and obviously the class
that gives name to the module itself:

```puppet
# == Class: <modulename>
#
# Full description of class <modulename> here.
#
# === Parameters
#
# Document parameters here.
#
# [*sample_parameter*]
#   Explanation of what this parameter affects and what it defaults to.
#   e.g. "Specify one or more upstream ntp servers as an array."
#
# === Variables
#
# Here you should define a list of variables that this module would require.
#
# [*sample_variable*]
#   Explanation of how this variable affects the funtion of this class and if it
#   has a default. e.g. "The parameter enc_ntp_servers must be set by the
#   External Node Classifier as a comma separated list of hostnames." (Note,
#   global variables should not be used in preference to class parameters  as of
#   Puppet 2.6.)
#
# === Examples
#
# === Authors
#
# Author Name <author@domain.com>
#
# === Copyright
#
# Copyright 2013 Your name here, unless otherwise noted.
#
class <modulename> {
}
```

## Ruby and Gemfile
We rely on functionality provided by a bunch of external gems. In this cases 
(i.e. always), I manage Ruby projects with 
[rbenv](https://github.com/sstephenson/rbenv/) and 
[Bundler](http://gembundler.com/), so every folder is perfectly isolated from 
the others.

I won't go into installing **rbenv** and the **ruby-build** plugin. You can
read the excellent documentation on project's homepage. Instead, let's declare
which version of Ruby we want to use:
```bash
rbenv local 1.9.3-p374
```

Then, assuming gem `bundler` is already installed, let's setup project
dependencies in the `Gemfile`:
```ruby
source :rubygems
puppetversion = ENV.key?('PUPPET_VERSION') ? "= #{ENV['PUPPET_VERSION']}" : ">= 3.0"

gem 'rake'

group :development do
  gem 'puppet', puppetversion
end

group :test do
  gem 'rspec'
  gem 'rspec-puppet'
  gem 'rspec-hiera-puppet'
  gem 'puppet-lint'
  gem 'ci_reporter'
  gem 'puppetlabs_spec_helper'
end
```
This is what's in there:

* on **line 2**, we set a variable to require the correct puppet version, depending
  on which one we want to develop upon. This variable is tweakable from the
  external environment; this is useful e.g. if we want to perform a matrix
  build of the module, checking if code works for different versions of Puppet
* `rspec-puppet` provides a set of helpers for writing tests against Puppet
  code
* `rspec-hiera-puppet` allows us to use/test Hiera in our tests
* `puppet-lint` provides a Rake task to check if Puppet code adhere to
  Puppetlabs guidelines
* `ci_reporter` provides an helper to generate reports that can be read from
  Jenkins
* `puppetlabs_spec_helper` provides a set of helpers to handle initialization
  of Puppet environment when testing.


At this point we can proceed to installing the module's dependencies with:

```bash
bundle install --path vendor/bundle
```

## Rspec initialization
The next step is to setup the directory that will hold our test code. Luckily,
the excellent **rspec-puppet** tool can help us a lot:

```bash
bundle exec rspec-puppet-init
```

In particular, the command above will take care of symlinking the folders in
the module's root into `spec/fixtures/modules/<modulename>`, looking at the
main module's manifest to find the correct name to use. It will also setup
a basic `spec_helper` and `Rakefile`; the latter is the first step into CI
integration.

At this point we need to tweak the configuration in a couple of points.  
First thing first, as I explain in [this Gist](https://gist.github.com/stefanozanella/4190920),
if you want to test a Puppet declaration that uses a function coming from an
external module (as those in **stdlib**), you need to have the functions into
the `LOAD_PATH` before running the tests. So, put the following line on top of
your `spec_helper.rb`:

```ruby
$:.unshift *Dir.glob(File.expand_path('../fixtures/modules/*/lib', __FILE__))
```

Also, we want to use 
[Puppetlabs' _spec_helper_](https://github.com/puppetlabs/puppetlabs_spec_helper)
so we can avoid setting up Puppet state for testing. Following the instruction
on the project's home page, we again update `spec_helper` as follows:
```ruby
require 'puppetlabs_spec_helper/module_spec_helper'
```

## Rakefile
Next, we put our attention to the `Rakefile`. We want to:

 * include the set of tasks provided by `puppetlabs_spec_helper`
 * configure the `ci_reporter` gem to report our test results to Jenkins
 * add a task to perform syntax-checking on the module's code

The first step is the simplest; it's as simple as adding the following line to
the `Rakefile`:
```ruby
require 'puppetlabs_spec_helper/rake_tasks'
```

This will make several tasks available to rake. One of these is `lint`, which
we want to use to check if our code is best-practices compliant; unfortunately,
the standard way it works is to check every `*.pp` file into the codebase. This
means it will also check manifests e.g. from the Puppet codebase we have into
`vendor`. You can easily understand that this way everything is going berserk.  
To solve this problem we can tweak the configuration of `puppet-lint`, changing
the paths it will look into:
```ruby
# Configured to be recognizable by Jenkins warnings plugin
require 'puppet-lint/tasks/puppet-lint'
PuppetLint.configuration.ignore_paths = ["vendor/**/*.pp", "spec/**/*.pp"]
PuppetLint.configuration.log_format =
  "%{path}:%{linenumber}:%{check}:%{KIND}:%{message}"
```
Contextually, we also set the format of the output so Jenkins can recognize
what the lint tool is reporting and show it in its awesome UI.

Next, we want to report on Jenkins how many tests there are in our codebase and
how many of them are broken. Jenkins can store the history of this metric and
show it in a nice graph on the build job dashboard. This task is performed by
the `ci_reporter` gem, which can be configured as follows:
```ruby
# Used by Jenkins to show tests report.
require 'ci/reporter/rake/rspec'
ENV['CI_REPORTS'] = 'reports'
```
Last, before checking if Puppet code adhere to the guidelines, we want to check if
it is actually syntactically correct. To do this, Puppet's CLI tool provides a
**parser** face that does just that. Since I prefer, where possibile, to do things
in code instead of invoking bash commands in Rakefiles (which in turn call code :), 
the Puppet face module must be loaded, and then the parser face can be called
asking for code validation. The rake task to perform this check can be written
as follows:
```ruby
# Invoked by Jenkins to validate manifest syntax
require 'puppet/face'
desc "Perform puppet parser's validation on manifests"
task :validate do
  Puppet::Face[:parser, '0.0.1'].validate(FileList['**/*.pp'].exclude('vendor/**/*.pp', 'spec/**/*.pp').join())
end
```

To finish with the Rakefile, we remove the code installed by default by
`rspec-puppet`, which is obsolete given we included Puppetlabs' helper. And,
since I'm fucking lazy, let's add a default task that performs validation,
linting and testing in a single command.  
The final `Rakefile` looks like:
```ruby
require 'rake'
require 'puppetlabs_spec_helper/rake_tasks'

# Configured to be recognizable by Jenkins warnings plugin
require 'puppet-lint/tasks/puppet-lint'
PuppetLint.configuration.ignore_paths = ["vendor/**/*.pp", "spec/**/*.pp"]
PuppetLint.configuration.log_format =
  "%{path}:%{linenumber}:%{check}:%{KIND}:%{message}"

# Used by Jenkins to show tests report.
require 'ci/reporter/rake/rspec'
ENV['CI_REPORTS'] = 'reports'

# Invoked by Jenkins to validate manifest syntax
require 'puppet/face'
desc "Perform puppet parser's validation on manifests"
task :validate do
  Puppet::Face[:parser, '0.0.1'].validate(FileList['**/*.pp'].exclude('vendor/**/*.pp', 'spec/**/*.pp').join())
end

desc "Run puppet-lint and rspec puppet in sequence"
task :default => [:lint, :validate, :spec]
```

## Fixtures 
As stated in 
[this post](https://puppetlabs.com/blog/the-next-generation-of-puppet-module-testing/)
from PuppetLabs, we need to setup a file that declares the fixtures we want to
have in place when testing our codebase. Let's pretend the module has a
dependency on PuppetLabs' `stdlib` (which is what happens in most cases, if,
for example, you're using validation - and you **DO** validate your 
parameters, right?); we need to setup a `.fixtures.yml` file that looks like
this:
```yaml
fixtures:
  repositories:
    stdlib: git://github.com/puppetlabs/puppetlabs-stdlib.git
  symlink:
    <modulename>: "#{source_dir}"
```
Remember that this file needs to be updated whenever module's dependencies
change.

## Modulefile
In the hope that someday Puppetlabs finally releases an API/client library to
publish modules to the Forge (which I think should be very soon from now),
let's create a `Modulefile` that describes the module and its dependencies.
This will also be useful to instruct `librarian-puppet` about the modules it
effectively needs to install.

```
name    '<author>-<modulename>'
version '0.0.1'
source 'UNKNOWN'
author '<author>'
license 'Apache License, Version 2.0'
summary '...'
description '.......'
project_page 'UNKNOWN'

## Add dependencies, if any:
dependency 'puppetlabs/stdlib', '>= 3.2.0'
```

## README
It's also good practice to add a `README`, as explained in 
[this must-read](https://puppetlabs.com/blog/writing-great-modules-an-introduction/)
from Puppetlabs itself. I won't go into details here, since apart from a common
layout, information in this file vary greatly depending on the project.  
The only thing I feel to say is quoting the suggestion in that same article
of reading the readme of Carl Caum's 
[Bacula module](http://github.com/puppetlabs/puppetlabs-bacula); that explains
pretty everything.

## Smoke tests
If you feel really cool, you can also add a smoke test into the `tests` folder,
which is basically a working manifest that can be called directly with `puppet
apply --noop`. If you're using the `module` face of the `puppet` command, it
will automatically create a `init.pp` that includes the main module's class.
Just, pay attention to the fact that you would need to keep this manifests
up-to-date with your code (e.g. setting proper parameters and instantiating
other resources that participate in the definition of a real, working
use-case).

## What happened to Git?
Till now, I completely left out versioning this skeleton project. This is
because I didn't find yet my preferred way to properly handle versioning and easily
enable to bootstrap new modules based on this skeleton. I don't exclude this
way could be putting everything in a gem or rake task or bash script that can bootstrap a
directory where a module is to be developed. Will see what the immediate future
will suggest. For now, please allow me not to give an answer on this subject.

## Additional resources
You can find additional information about testing Puppet code in
[these](https://puppetlabs.com/blog/the-next-generation-of-puppet-module-testing/)
[articles](https://puppetlabs.com/blog/test-driven-development-with-puppet/) 
from 
[Puppet Labs](https://puppetlabs.com/blog/verifying-puppet-checking-syntax-and-writing-automated-tests/).
Note that few things, like, Cucumber for Puppet, are considered deprecated; 
nevertheless, they're worth reading if you care about quality.
