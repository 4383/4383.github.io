---
layout: post
title:  "How to patch openstack tripelo"
description: "How to patch openstack tripleo and especially your undercloud to test your changes"
date:   2019-7-23 16:31:00 +0100
image: tripleo.png
categories: [openstack, tripleo, gerrit, patch, heat-template]
---
## Introduction

The goal of this post is to explain how to patch an instance of openstack
tripleo with a git patch set of changes.

You can test local changes or a specific patch set available on
gerrit or github by example.

During this walkthrough we will using an OSP15 freshly deployed by using
[infrared](https://infrared.readthedocs.io/en/stable/index.html) (cf. [my previous post about how to deploy by using infrared]({% post_url 2019-2-5-prepare-environment-to-use-red-hat-infrared %})).

To explain how things works we will using these 2 patches set:
- [https://review.opendev.org/#/c/668862/](https://review.opendev.org/#/c/668862/)
- [https://review.opendev.org/#/c/671254/](https://review.opendev.org/#/c/671254/)

So, we want to deploy the 2 previous patches on an OSP15 and after this
we want to redeploy your undercloud to apply our changes on your instance.

This walkthrough I will give you a tiny idea of:
- how to patch an undercloud already deployed
- how to test patches on an environment (during development period by example)
- how openstack deployment mechanismes works (the puppet part)

The idea behind the applying of these changes is to switch from an
apache MPM engine to another and to redeploy our undercloud after this to check
if the MPM engine in use is an issue with eventlet and green threads.

We will switch from the apache MPM prefork module who has the default engine
currently in use in the whole openstack, to the event module who can work with
async libraries and modern kernel features like epoll.

For further reading about the apache MPM module switching part you can
[read my previous blog post]({% post_url 2019-7-15-how-nova-consume-oslo-messaging %})
where I explained the issue and the reasons behind these changes.

## Who this article is for?

For developers and engineers who wants to understand how to debug openstack and
especially how to patch a running instance of openstack without spending to
redeploy everything.

## Requirements

- An OSP15 already deployed
- An ssh access to your undercloud instance

### Patch the undercloud deployment templates

First, connect you to your undercloud instance as a `stack` user.

All that you need to patch is hosted in the `usr/share` directory.

Then become `root`.

We will apply [this patch](https://review.opendev.org/#/c/671254/) first to
fix an issue who need to be fixed first
to let's our apply our changes properly in a second time.

These changes are related to the [tripleo-heat-templates](https://github.com/openstack/tripleo-heat-templates/) project.

```
$ sudo su -
# cd /usr/share/openstack-tripleo-heat-templates/
# curl 'https://review.opendev.org/changes/671254/revisions/64b57d8459c2912a7bb9c4eedf932bbe61db6a9a/patch?download' | base64 -d | patch -p1 -d .
```

Now we want to applying the changes related to the new apache MPM engine to use
aka the [MPM `event` module](https://httpd.apache.org/docs/2.4/mod/event.html).

To switch from an engine to another we will use
[this patch](https://review.opendev.org/#/c/668862/).

This patch is for testing purpose only, [the final implementation is a little
bit different](https://review.opendev.org/#/c/671321/),
because we don't want to force the switching, we want to let
users choose the right engine to use for their services.

Also in the final implementation we want to prevent updates and upgrades
impacts.

In my context, my main goal is to test if using the `event` engine prevent
the heartbeat and eventlet issues so I will applied this engine everywhere
on each service.

This patch correspond to changes on the [`puppet-tripleo` project](https://github.com/openstack/puppet-tripleo):
```
$ sudo su -
# cd /usr/share/openstack-puppet/modules/tripleo/
# curl 'https://review.opendev.org/changes/668862/revisions/dec62a3341cfeca0ed3904e778095dfaf63b7f47/patch?download' | base64 -d | patch -p1 -d .
```

## Redploy the undercloud

Now our changes are applied so we can redeploy our undercloud.

As a `stack` user on our undercloud we will run:
```shell
$ ./undercloud_deploy.sh &
$ tail -f undercloud_install.log 
```

If everything work fine you will see similar output in your shell after a
few moment:
```shell
##########################################################

The Undercloud has been successfully installed.

Useful files:

Password file is at ~/undercloud-passwords.conf
The stackrc file is at ~/stackrc

Use these files to interact with OpenStack services, and
ensure they are secured.

##########################################################
```

During the deployment we have use our patched files and config so normally
we now use the MPM `event` instead of using the MPM `prefork` module, let observe

## Check which MPM module we use now

We just redeploy our undercloud so all the running containers and services
have been reloaded with our newest configuration.

Let see that, first, check service uptime:
```shell
{% raw %}
$ sudo podman ps --filter name=api --format "{{.Status}} => {{.Names}}"
{% endraw %}
Up 3 hours ago => mistral_api
Up 3 hours ago => ironic_api
Up 3 hours ago => nova_api
Up 3 hours ago => glance_api
Up 3 hours ago => nova_api_cron
Up 3 hours ago => heat_api_cron
Up 3 hours ago => heat_api_cfn
...
```

You can see that all the openstack APIs have been restarted 3 hours ago, the
time when I had redeployed my undercloud.

Then check which apache MPM engine is in use in the nova-api container:
```shell
$ # as the stack user execute to observe the apache loaded modules
$ sudo podman exec -it nova_api apachectl -M
Loaded Modules:
 core_module (static)
 so_module (static)
 http_module (static)
 ...
 dav_module (shared)
 dav_fs_module (shared)
 deflate_module (shared)
 dir_module (shared)
 env_module (shared)
 mpm_event_module (shared)
 ...
 setenvif_module (shared)
 socache_shmcb_module (shared)
 speling_module (shared)
 ssl_module (shared)
 status_module (shared)
 substitute_module (shared)
 suexec_module (shared)
 systemd_module (shared)
 unixd_module (shared)
 usertrack_module (shared)
 version_module (shared)
 vhost_alias_module (sh
 wsgi_module (shared)
```

We can see that `mpm_event_module` is in list so is in use in our apache config.

We can also check if the `prefork` module is loaded or not:
```shell
$ sudo podman exec -it nova_api apachectl -M | grep -E "event|prefork"
 mpm_event_module (shared)
```

The prefork module isn't loaded, sound good!

### Conclusion

It's really easy to test things and changes without spend time to redeploy all
the whole openstack by using infrared.

In this article we have used patches downloaded from the
[openstack gerrit](http://review.openstack.org/) but you can also apply local changes
by simply replacing the `curl|patch` commands with the right command statements.

Patches can also be hosted on github or on any dist-git of your choice.

Now that my undercloud is patched I need to check the behaviour of
the AMQP heartbeat used in a monkey patched nova-api under apache MPM event and
mod_wsgi to see if green threads works better.

I hope you enjoy reading this article and I hope some of these informations
can be useful for you.
