---
layout: post
title:  "How to run tests locally for openstack puppet projects"
description: "Configure and run an rspec lab to run your tests"
date:   2019-11-25 18:53:00 +0100
image: puppet.png
categories: [puppet, rspec, openstack, tests]
---
## Introduction
Some deployment parts of openstack are based on [puppet](https://puppet.com/)
and so openstack host a bunch of
[openstack puppet based projects](https://github.com/search?q=org%3Aopenstack+puppet&unscoped_q=puppet).

These projects are an effort by the Openstack infrastructure team to provide
continuous integration testing and code review for OpenStack and OpenStack
community projects as part of the core software.

Each project is a thorough attempt to make Puppet capable of managing the
entirety of specific parts, like, nova, oslo, neutron, etc.

Each time you contribute to these projects you need to ensure yourself that
your features works and that your changes fit well.

Openstack puppet based projects' tests are designed by using the
[rspec-puppet framework](https://rspec-puppet.com/). 

However,`rspec-puppet` need several dependencies on your environment or your
laptop, so it can be difficult to setup a puppet environment for
testing purpose, especially when you are not a regular contributor
to these puppet projects.

In this blog post we will show how to test your changes in a merely manner
by using the [rspec-container](https://github.com/cjeanner/rspec-container).

The `rspec-container` project provide to you a preconfigured environment via
a pre designed container who will avoid you to spend time to setup this and
to be more focused on your tests and features.

## Prerequisites

To run this recipe you need to install the following tools:
- docker
- docker-compose
- git

## Setup your lab

Here we start by setup our rspec lab.

First, clone the `rspec-container` project:

```sh
$ git clone git@github.com/cjeanner/rspec-container
```

Note that `rspec-container` will mount some default path and dotfiles:
- `~/work`
- `~/.vimrc`
- `~/.gitconfig`

These mounts are defined in the [docker-compose file](https://github.com/cjeanner/rspec-container/blob/master/docker-compose.yaml#L11,L14).

The `work` directory normally contains your repositories and projects.

If like me you doesn't have the `work` directory on your laptop and you don't
want to edit the docker compose file at each run, you can create a symlink who
narrow your own development directory.

By example in my case it looks like to:

```sh
$ ln -s ~/dev ~/work
```

Then now you can start your lab and enter on:

```sh
$ docker-compose up &
Creating rspec-container_rspec_1 ... done
Attaching to rspec-container_rspec_1
$ docker exec -it rspec-container_rspec_1 /bin/bash
```

## Test your projects

As I'm an openstack oslo core developer, I will use the
[puppet-oslo](https://github.com/openstack/puppet-oslo) project as an
example here.

Consider that we are in connected to our lab, and consider that we have
already cloned the `puppet-oslo` project under `~/work`, so our environment
look like:

```sh
$ ls -la ~/work
drwxrwxr-x. 12 herve herve 4096 Nov 25 10:36 puppet-oslo
```

Then go to your project directory (or clone your own if needed):

```sh
$ cd work/puppet-oslo
$ bundle install
Fetching gem metadata from https://rubygems.org/.........
Fetching version metadata from https://rubygems.org/...
Fetching dependency metadata from https://rubygems.org/..
Fetching https://opendev.org/openstack/puppet-openstack_spec_helper
Installing rake 13.0.1
Installing CFPropertyList 2.3.6
Installing concurrent-ruby 1.1.5
Installing minitest 5.13.0
Installing thread_safe 0.3.6
...
Using puppet-openstack_spec_helper 16.0.0 from https://opendev.org/openstack/puppet-openstack_spec_helper (at master@d38ed71)
Bundle complete! 3 Gemfile dependencies, 139 gems now installed.
Use `bundle show [gemname]` to see where a bundled gem is installed.
$ # be sure to add binaries to your env
$ export PATH=$PATH:$HOME/bin:/var/lib/gems/1.8/bin
$ bundle exec rake spec # depend on your project but sometime `spec` will be named `rspec`
rm -rf openstack/puppet-openstack-integration
git clone https://opendev.org/openstack/puppet-openstack-integration openstack/puppet-openstack-integration
Cloning into 'openstack/puppet-openstack-integration'...
remote: Enumerating objects: 7326, done.
remote: Counting objects: 100% (7326/7326), done.
remote: Compressing objects: 100% (3291/3291), done.
remote: Total 7326 (delta 5063), reused 6193 (delta 4005)
Receiving objects: 100% (7326/7326), 1.27 MiB | 415.00 KiB/s, done.
Resolving deltas: 100% (5063/5063), done.
env PUPPETFILE_DIR=/home/hberaud/work/redhat/upstream/openstack/puppet-oslo/spec/fixtures/modules ZUUL_BRANCH= bash openstack/puppet-openstack-integration/install_modules_unit.sh
...
Finished in 1 minute 51.47 seconds (files took 2 minutes 4.5 seconds to load)
1050 examples, 0 failures 
```

If everything work fine you will see a message who looks like to something like:

```
Finished in 1 minute 51.47 seconds (files took 2 minutes 4.5 seconds to load)
1050 examples, 0 failures
```

Now that your environment works and can spend time to develop your tests and
do some back and forth with your patches.
