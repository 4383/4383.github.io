---
layout: post
title:  "Apache MPM prefork - AMQP heartbeat and eventlet issue"
description: "How openstack service like nova-api consume oslo.messaging. Walkthrough to understand how openstack services consume oslo.messaging and the rabbitmq driver and to understand the root cause of the AMQP heartbeat/eventlet issue under apache MPM prefork"
date:   2019-7-15 16:31:00 +0100
image: oslo-logo.png
categories: [openstack, nova, oslo.messaging, rabbitmq, driver]
---
## Introduction

The goal of this post is to centralize pieces of my analyze and of the original
puzzle, related to an openstack issue with the AMQP heartbeat on nova-api
under apache mod_wsgi.

I this post I want to show you:
- a brief description of the original issue and possibly why we facing this issue
- an analyze of the root cause and why the default apache MPM `prefork` module in use with eventlet is an issue
- how openstack services consume oslo.messaging and especially how they use the rabbitmq driver.
- some POC to test different behaviours

These info can be useful for other persons so I want to share these
with you.

My walkthrough (cf. section bellow) had help me to understand how things works together, so I think
it can be useful to also add it there even if it's a little bit out of the scope
of the fix and the real issue.

I'm not a nova expert and maybe I'm wrong on some points, so, if you see errors
in this walkthrough do not hesitate to fix it and to
[submit a pull request](https://github.com/4383/4383.github.io).

## Who this article is for?

For developers and engineers who wants to understand why apache with eventlet
can be an issue in some situations.

You don't need to be developer on openstack, if you want to use eventlet through
a HTTP restful API this article can be useful for you to help you to understand
the dependencies in your stack and the execution model behind your choices.

## The original issue

We facing an issue with the rabbitmq driver heartbeat
under [apache MPM `prefork` module](https://httpd.apache.org/docs/2.4/mod/prefork.html)
and with [`mod_wsgi`](https://modwsgi.readthedocs.io/en/develop/) when nova-api [monkey
patch the stdlib by using eventlet]((https://eventlet.net/doc/patching.html).

[nova-api calling eventlet.`monkey_patch()`](https://github.com/openstack/nova/commit/3c5e2b0e9fac985294a949852bb8c83d4ed77e04) when it runs under mod_wsgi.

This impacts the AMQP heartbeat thread, which is meant to be a native thread.
Instead of checking AMQP sockets every 15s, It is now suspended and resumed by
eventlet.

However, resuming greenthreads can take a very long time if mod_wsgi isn't
processing traffic regularly, which can cause rabbitmq to close the AMQP
connection.

Here is a list of related issues:
- [[OSP15][deployment] AMQP heartbeat thread missing heartbeats when running under nova_api - BZ1711794](https://bugzilla.redhat.com/show_bug.cgi?id=1711794)
- [eventlet monkey-patching breaks AMQP heartbeat on uWSGI - LP1825584](https://bugs.launchpad.net/nova/+bug/1825584)
- [systemd timers for podman healthchecks are too high, break AMQP healthchecks - LP1826281](https://bugs.launchpad.net/tripleo/+bug/1826281)
- [Use oslo_rootwrap subprocess module in order to gain proper eventlet awareness](https://review.opendev.org/#/c/656901/)
- [futurist threadpoolexecutor](https://review.opendev.org/#/c/650172/)
- [Do not monkey_patch nova_api running under mod_wsgi](https://review.opendev.org/#/c/657168/)
- [nova eventlet ML discussion](http://lists.openstack.org/pipermail/openstack-discuss/2019-April/005310.html)
- [[oslo][oslo-messaging][nova] Stein nova-api AMQP issue running under uWSGI](http://lists.openstack.org/pipermail/openstack-discuss/2019-May/005822.html)
- [eventlet best practices on openstack](ttps://review.opendev.org/#/c/154642/)
- [nova heartbeat and eventlet number of threads](https://bugs.launchpad.net/nova/+bug/1829062)

## The Origins of the RabbitMQ heartbeat

The [RabbitMQ heartbeat was introduced few years ago](https://bugs.launchpad.net/nova/+bug/856764)
to keep connections from
various components into RabbitMQ alive. In some situations, by example by
placing a stateful firewall between the connection could result in idle
connection being terminated without either endpoint being aware.

## Root Cause

The oslo.messaging RabbitMQ driver and especially the heartbeat
suffer to inherit the execution model of the service which consume him.

In this scenario nova-api need green threads to manage cells and edge
features so nova-api monkey patch the stdlib to obtain async features,
and the oslo.messaging rabbitmq driver endure these changes.

On openstack The default apache MPM module in use is the `prefork` module.

I think the main issue here is that nova-api want async and use eventlet green
threads to obtain it, because, eventlet is based on [epoll](https://github.com/eventlet/eventlet/blame/master/README.rst#L3)
or [libevent](https://libevent.org/), in an environment based on
apache MPM `prefork` module who doesn't support epoll and recent kernel
features.

The MPM `prefork` apache module is appropriate for sites that
need [to avoid threading for compatibility with non-thread-safe
libraries](https://httpd.apache.org/docs/2.4/fr/mod/prefork.html).

So we suspect that the apache MPM engine (`prefork`) in use here is also a part
of the problem.

## solutions

We have 2 possible solutions to fix the issue:
- fixing the oslo.messaging RabbitMQ driver by using native python thread for the heartbeat
- change the apache MPM module in use from `prefork` to `event`

### The oslo.messaging fix

To avoid similar issue we want to allow user to isolate the heartbeat
execution model from the parent process inherited execution model by passing the
`heartbeat_in_pthread` option through the driver config.

While we use MPM `prefork` we want to avoid to use libevent and epoll.

If the `heartbeat_in_pthread` option is given we want to force to use the
python stdlib threading module to run the
rabbitmq heartbeat to avoid issue related to a non "standard"
environment. I mean "standard" because async features isn't the default
config in mostly case, starting by apache which define `prefork` is the
default engine.

This is an experimental feature, we can help us to ensure to run heartbeat
through a classical python thread
not build over epoll and libevent to avoid issue with environment who
doesn't support these kernel features.

Proposed fix:
- https://review.opendev.org/#/c/663074/

### The apache MPM fix

We will try to switch [4][5] from the apache `prefork` module to the `event`
module which support non blocking sockets, use modern kernel features
like epoll through [APR](https://httpd.apache.org/docs/2.4/fr/glossary.html#apr).

Proposed fixes:
- https://review.opendev.org/#/c/668862/
- https://review.opendev.org/#/c/669178/

These changes are not fully tested yet with the oslo.messaging rabbitmq
heartbeat. I need to realize some tests with a non patched version of
oslo.messaging to observe if it work better with eventlet under apache and
mod_wsgi.

Possibly if these changes work as expected the oslo.messaging fix
(cf. previous section) will become obsolete and we could restore the original
behaviour. An inhereted parent execution model would not be an issue for
oslo.messaging.

I will update this section when I'll more info to share with you about this.

## Walkthrough - how nova-api use oslo.messaging

To see how things works in openstack we will do a walkthrough to inspect
how the nova-api consume and use the oslo.messaging rabbitmq driver.

We will start by analyze how openstack run the nova-api, the different
approaches to launch the service (puppet, ansible), in a second time we will
observe how the WSGI application for the nova-api work,
and to finish we will see how nova-api
established the connections with the rabbitmq driver and how the health check
heartbeat will be launched.

I've initiate this walkthrough to better understand how openstack service works
and to try to fix an [oslo.messaging issue related to the rabbitmq driver](https://bugs.launchpad.net/nova/+bug/1825584) when
we run it through nova in an [eventlet monkey patched environment](https://eventlet.net/doc/patching.html)
under [apache mod_wsgi](https://modwsgi.readthedocs.io/en/develop/) (cf. the previous described issue).

### The puppet part

The [puppet team](https://wiki.openstack.org/wiki/Puppet) provide a [puppet project dedicated to nova](https://github.com/openstack/puppet-nova/)
who define how to run nova under an apache environment. Also it's important to note
that other deployments systems exists and are in use on openstack, like ansible through
the [openstack ansible project](https://docs.openstack.org/project-deploy-guide/openstack-ansible/latest/)
who provide [a different approach to deploy nova](https://github.com/openstack/openstack-ansible-os_nova).
There are many others deployment systems in use too (Chef, Salt, etc.).

This puppet project will configure [apache to run nova](https://github.com/openstack/puppet-nova/blob/master/manifests/wsgi/apache_api.pp#L158)

It is the `script_path` is used for the `wsgi_script_dir` in configuring the
[`httpd conf`](https://github.com/openstack/puppet-nova/blob/master/manifests/wsgi/apache_api.pp#L158).
In the end its all used by https://github.com/openstack/puppet-openstacklib/blob/master/manifests/wsgi/apache.pp to configure the wsgi settings of the vhost

The wsgi [nova entry_point to use to start the nova wsgi application is defined here and the apache vhost file to use too](https://github.com/openstack/puppet-nova/blob/29d307bce168a39477953a856382c2402ac1ff77/spec/classes/nova_wsgi_apache_api_spec.rb#L132,L135)

We can observe that this puppet script define the following configuration `api_wsgi_script_source => '/usr/bin/nova-api-wsgi`.

Nova will be launched by calling the wsgi script available at `/usr/bin/nova-api-wsgi`.

The nova-api will be runned under the apache MPM `prefork` module, like described previously.

This wsgi scripts is generated by [pbr](https://docs.openstack.org/pbr/latest/user/using.html#entry-points)
and [setuptools](https://setuptools.readthedocs.io/en/latest/setuptools.html?highlight=entry_points) during the nova install.

Nova [define it in its own setup.cfg file](https://github.com/openstack/nova/blob/master/setup.cfg#L76).

This entry point will when it's called while trigger `nova.api.openstack.compute.wsgi:init_application`
[who correspond to the initialization of the nova wsgi application](https://github.com/openstack/nova/blob/master/nova/api/openstack/wsgi_app.py#L74).

To implement the [Python Web Server Gateway Interface (WSGI)(PEP 3333)](https://www.python.org/dev/peps/pep-3333/) nova
use the [`paste.deploy` python module](https://github.com/Pylons/pastedeploy).

### Generate the wsgi apps by using Paste Deploy

Paste deploy was at the origin a submodule of the [paste module](https://github.com/cdent/paste/).

Basically at some point in the past people realized that `paste.deploy` was
one of the main useful parts of paste and they didn't want all the rest of
paste, so it was extracted.

As paste and paste.deploy both got "old" maintenance sort of diverged.
They are still maintained, but as separate packages.

Even if the [Paste module](https://github.com/cdent/paste) seems not used here
I will describe some `paste` specific behaviours and especially
[concerning requests handling, threads, and the python interpreter life cycle](https://github.com/cdent/paste/blob/5a542da992618e30a508f7f03259f63cf2ee1ceb/docs/paste-httpserver-threadpool.txt#L40),
which I guess we need to take care to really undestand the eventlet issue and green threads for the heartbeat.

Indeed, to avoid issue with requests management (freeze, memory usage, etc...),
`paste` manage threads with 3 states:
- idle (waiting for a request)
- working (on a request)
- should die (a thread that should die for some reasons)

If a `paste` thread initiate the oslo.messaging AMQP heartbeat who will run in
an async way by using green thread maybe in this case the parent thread consider that
it can become idle for some reasons and the main reason is that this thread
(heartbeat) is not a blocking thread.

In parallel, uwsgi (not used here but the heartbeat seems occur within too)
support for `paste` facing some [issues in multiple process/workers mode](https://uwsgi-docs.readthedocs.io/en/latest/Python.html#paste-support)

On the `paste.deploy` side [`nova` call](https://github.com/openstack/nova/blob/master/nova/api/openstack/wsgi_app.py#L99)
the [`loadapp`](https://github.com/Pylons/pastedeploy/blob/master/paste/deploy/loadwsgi.py#L252) method to serve the nova-api.

Paste Deployment is a system for finding and configuring WSGI applications
and servers. 
For WSGI application consumers it provides a single, simple function (loadapp)
for loading a WSGI application from a configuration file or a Python Egg.
For WSGI application providers it only asks for a single, simple entry point
to your application, so that application users don't need to be exposed to the
implementation details of your application.

The nova service call the `loadapp` method by passing available configurations.
The configuration seems to be designed by following the
[nova configuration guide](https://docs.openstack.org/nova/latest/configuration/index.html).

A sample configuration example for nova [is available online](https://docs.openstack.org/nova/latest/configuration/sample-config.html) (cf. the wsgi part).

Paste deploy don't seem to define specific behaviours related to threads management.
It only seem to help to define application parts and related url/uri, database url, etc...

### Running the WSGI app and services at start

Well, now we will continue to follow the nova code.

During the init of the nova-api application, nova will try to setup some services.
These services are setup by using the [`_setup_service` method](https://github.com/openstack/nova/blob/master/nova/api/openstack/wsgi_app.py#L42).

Services to setup are defined in a [service manager](https://github.com/openstack/nova/blob/013aa1915c79cfcb90c4333ce1e16b3c40f16be8/nova/service.py#L55,L62).

Nova manage services by using a [dedicated service module](https://github.com/openstack/nova/blob/f2b96588efa931fa10b188f0602738c484b965ed/nova/objects/service.py).

They services seems to be also retrieving by [querying the database](https://github.com/openstack/nova/blob/f2b96588efa931fa10b188f0602738c484b965ed/nova/objects/service.py#L321)
by [using the previous](https://github.com/openstack/nova/blob/master/nova/api/openstack/wsgi_app.py#L46) definied service module.

To continue this inspection of nova we will now become focused on the [`ConsoleAuthManager`](https://github.com/openstack/nova/blob/6ac15734b9678bfb876e7dfa6502a13d1fe2ee40/nova/consoleauth/manager.py#L38) module.
This class will instanciate a [`compute_rpcapi.ComputeAPI` object](https://github.com/openstack/nova/blob/6ac15734b9678bfb876e7dfa6502a13d1fe2ee40/nova/consoleauth/manager.py#L48)
This object ([`ComputeAPI`](https://github.com/openstack/nova/blob/51ed40a6a5fc046cef35337980a1fc5ad704a421/nova/compute/rpcapi.py#L73)) define
a [`router` method](https://github.com/openstack/nova/blob/51ed40a6a5fc046cef35337980a1fc5ad704a421/nova/compute/rpcapi.py#L383) who will return
a [rpc client](https://github.com/openstack/nova/blob/51ed40a6a5fc046cef35337980a1fc5ad704a421/nova/compute/rpcapi.py#L405).

The returned [RPC client](https://github.com/openstack/nova/blob/244c9240671d98b0df25b0ad0795b5de0c0c422c/nova/rpc.py#L208) returned by the [nova rpc module](https://github.com/openstack/nova/blob/master/nova/rpc.py) is an [oslo.messaging rpc client](https://github.com/openstack/nova/blob/master/nova/rpc.py#L18)

The instantiated oslo.messaging [`RPCClient`](https://github.com/openstack/oslo.messaging/blob/master/oslo_messaging/rpc/client.py#L239) 

The transport layer is defined by [nova](https://github.com/openstack/nova/blob/244c9240671d98b0df25b0ad0795b5de0c0c422c/nova/rpc.py#L69) and [it will be retrieved](https://github.com/openstack/nova/blob/244c9240671d98b0df25b0ad0795b5de0c0c422c/nova/rpc.py#L259) by using [the oslo.messaging mechanismes based on the config and the url](https://github.com/openstack/oslo.messaging/blob/master/oslo_messaging/transport.py#L218) and the oslo.messaging [defined drivers](https://github.com/openstack/oslo.messaging/blob/master/setup.cfg#L38,L50) by using [stevedore](https://github.com/openstack/oslo.messaging/blob/master/oslo_messaging/transport.py#L204).

In our case we will instantiate a [`RabbitDriver`](https://github.com/openstack/oslo.messaging/blob/master/oslo_messaging/_drivers/impl_rabbit.py#L1266).

The used driver will initiate a [connection pool](https://github.com/openstack/oslo.messaging/blob/master/oslo_messaging/_drivers/impl_rabbit.py#L1299,L1301)
by using the [Connection class](https://github.com/openstack/oslo.messaging/blob/master/oslo_messaging/_drivers/impl_rabbit.py#L417) defined in the driver.

The oslo.messaging connection pool module [will create the connection](https://github.com/openstack/oslo.messaging/blob/master/oslo_messaging/_drivers/pool.py#L144)
and also start the healt check mechanism by [triggering the heartbeat in a dedicated thread](Also://github.com/openstack/oslo.messaging/blob/master/oslo_messaging/_drivers/impl_rabbit.py#L897)

Also on the oslo.messaging we need to [take care about the connection class
execution model](https://github.com/openstack/oslo.messaging/blob/40c25c2bde6d2f5a756e7169060b7ce389caf174/oslo_messaging/_drivers/common.py#L369,L387).
Even if rabbit has only one Connection class,
this connection can be used for two purposes:
* wait and receive amqp messages (only do read stuffs on the socket)
* send messages to the broker (only do write stuffs on the socket)
The code inside a connection class is not concurrency safe.
Using one Connection class instance for doing both, will result
of eventlet complaining of multiple greenthreads that read/write the
same fd concurrently... because 'send' and 'listen' run in different
greenthread.
So, a connection cannot be shared between thread/greenthread and
this two variables permit to define the purpose of the connection
to allow drivers to add special handling if needed (like heatbeat).
amqp drivers create 3 kind of connections:
* driver.listen*(): each call create a new 'PURPOSE_LISTEN' connection
* driver.send*(): a pool of 'PURPOSE_SEND' connections is used
* driver internally have another 'PURPOSE_LISTEN' connection dedicated
  to wait replies of rpc call

On an over hand the connection pool seems to define the [`_on_expire` event
listener](https://github.com/openstack/oslo.messaging/blob/044e6f20e65084f3c4ecc554672d3271b2a2acd3/oslo_messaging/_drivers/pool.py#L137).

This listener seems to be called when an:
> Idle connection has expired and been closed

The _Idle connection_ here seem to be the connection with the rabbitmq server (by example) who have expired.
Then the "thread safe" [`Pool`](https://github.com/openstack/oslo.messaging/blob/044e6f20e65084f3c4ecc554672d3271b2a2acd3/oslo_messaging/_drivers/pool.py#L40) mechanism,
Modelled after the eventlet.pools.Pool interface, but designed to be safe
when using native threads without the GIL, defined the `expire` method who
will clean the pool from the expired connections based on a ttl.

I think we need to take care about the previous mechanism due to the fact that
the connection and the heartbeat have been invoqued from the connection pool mechanism
(cf. previous lines about how the connection pool module start the health check)

The nova rpc module also define a [RPC server](https://github.com/openstack/nova/blob/244c9240671d98b0df25b0ad0795b5de0c0c422c/nova/rpc.py#L215) inherited from [oslo.messaging](https://github.com/openstack/nova/blob/244c9240671d98b0df25b0ad0795b5de0c0c422c/nova/rpc.py#L18)

There the used executor is an [`eventlet` executor](https://github.com/openstack/nova/blob/244c9240671d98b0df25b0ad0795b5de0c0c422c/nova/rpc.py#L226), so the initiated object will use eventlet and so the heartbeat thread will use a green thread.

The oslo.messaging instantiate [RPC server](https://github.com/openstack/nova/blob/master/nova/rpc.py#L223) (https://github.com/openstack/oslo.messaging/blob/40c25c2bde6d2f5a756e7169060b7ce389caf174/oslo_messaging/rpc/server.py#L190)

## POC

Some POC are available at:
https://github.com/4383/pyamqp-heartbeat/blob/master/POC.md

You can test behaviors between execution models and different versions of the
oslo.messaging code base (patched/non-patched).

These POC can be useful to compare the difference between using a native python
thread or an eventlet green thread under apache MPM `prefork`.

These changes don't reflect the latest changes and especially the
`heartbeat_in_pthread` option and the possibility to turn on/off the
feature. In other words on these POCs we always force to use pthreads
and these POCs allow you to compare the rabbitmq heartbeat connection
with pthread and green thread.

## Conclusion

Congretulation! You are now at the end of this article, you've read the most biggest part
of this post.

Execution environment can have impactes on services, applications, and libraries,
like the nova-api/oslo.messaging use case.

It's not a trivial thing to debug, I hope my article can help some of you.

Openstack is a little bit complicated to debug, to run a service we need to
instantiate many environments and apps like apache, mod_wsgi, etc...

With stack like this it can be difficult to determine which part introduce
the issue, and why the error occur.

I hope you appreciated read this article, don't hesitate to fix errors if you
see one. Don't hesitate to contact me for further discussions and questions.

You can find further resources and useful links bellow.

Cheers!

## References & further reading

- [openstack - eventlet to futurist blueprint](https://specs.openstack.org/openstack/oslo-specs/specs/liberty/adopt-futurist.html)
- [[OSP15][deployment] AMQP heartbeat thread missing heartbeats when running under nova_api - BZ1711794](https://bugzilla.redhat.com/show_bug.cgi?id=1711794)
- [eventlet monkey-patching breaks AMQP heartbeat on uWSGI - LP1825584](https://bugs.launchpad.net/nova/+bug/1825584)
- [systemd timers for podman healthchecks are too high, break AMQP healthchecks - LP1826281](https://bugs.launchpad.net/tripleo/+bug/1826281)
- [Use oslo_rootwrap subprocess module in order to gain proper eventlet awareness](https://review.opendev.org/#/c/656901/)
- [futurist threadpoolexecutor](https://review.opendev.org/#/c/650172/)
- [Do not monkey_patch nova_api running under mod_wsgi](https://review.opendev.org/#/c/657168/)
- [nova eventlet ML discussion](http://lists.openstack.org/pipermail/openstack-discuss/2019-April/005310.html)
- [[oslo][oslo-messaging][nova] Stein nova-api AMQP issue running under uWSGI](http://lists.openstack.org/pipermail/openstack-discuss/2019-May/005822.html)
- [eventlet best practices on openstack](ttps://review.opendev.org/#/c/154642/)
- [nova heartbeat and eventlet number of threads](https://bugs.launchpad.net/nova/+bug/1829062)
- [RabbitMQ connections lack heartbeat or TCP keepalives](https://bugs.launchpad.net/nova/+bug/856764)
- [A 'greenio' executor for oslo.messaging](https://blueprints.launchpad.net/oslo.messaging/+spec/greenio-executor)
- [usage of the messaging rpc_transport](https://github.com/openstack/nova/search?q=get_rpc_transport&type=Code)
- [the nova rpc layer init the RPCClient](https://github.com/openstack/nova/blob/244c9240671d98b0df25b0ad0795b5de0c0c422c/nova/rpc.py)
- [the RPC client (ClientRouter) is in use in the nova rpcapi](https://github.com/openstack/nova/search?q=ClientRouter&unscoped_q=ClientRouter)
- [source code where the ClientRouter is used by the nova rpcapi (class ComputeAPI)](https://github.com/openstack/nova/blob/4af8da5b0b832cac6669c4241867f97899643ccd/nova/compute/rpcapi.py#L409)
- [the nova compute API ^^^ is in use in the nova consoles managers](https://github.com/openstack/nova/search?q=ComputeAPI&unscoped_q=ComputeAPI)
- [source code of usage of the compute api in the managers (consoleauth [deprecated])](https://github.com/openstack/nova/blob/6ac15734b9678bfb876e7dfa6502a13d1fe2ee40/nova/consoleauth/manager.py#L48)
- [source code of usage of the compute api in the managers (console)](https://github.com/openstack/nova/blob/c6218428e9b29a2c52808ec7d27b4b21aadc0299/nova/console/manager.py#L48)
- [the nova service will use the managers](https://github.com/openstack/nova/blob/013aa1915c79cfcb90c4333ce1e16b3c40f16be8/nova/service.py#L55)
- [[schema] example of life cycle during of call of the nova-api who will use the consoleauth](https://docs.openstack.org/nova/pike/_images/SCH_5009_V00_NUAC-VNC_OpenStack.png)
- [nova api is a rest api](https://docs.openstack.org/nova/latest/contributor/api-ref-guideline.html)
- [nova api infos](https://docs.openstack.org/nova/latest/contributor/index.html#the-nova-api)
- [nova wsgi init application](https://github.com/openstack/nova/blob/master/nova/api/openstack/compute/wsgi.py)
- [nova wsgi setup service](https://github.com/openstack/nova/blob/master/nova/api/openstack/wsgi_app.py#L42)
- [nova wsgi start](https://github.com/openstack/nova/blob/master/nova/api/openstack/compute/wsgi.py)
- [nova use paste to return the wsgi application](https://pypi.org/project/Paste/)
- [nova apache conf via puppet](https://github.com/openstack/puppet-openstacklib/blob/master/manifests/wsgi/apache.pp)
- [nova apache conf via puppet](https://github.com/openstack/puppet-nova/blob/master/manifests/wsgi/apache_api.pp#L158)
- [monkey patch ASAP on nova - commit](https://github.com/openstack/nova/commit/3c5e2b0e9fac985294a949852bb8c83d4ed77e04)
- [pep3333 - Python Web Server Gateway Interface v1.0.1](https://www.python.org/dev/peps/pep-3333/)
- [pep3333 - thread support](https://www.python.org/dev/peps/pep-3333/#thread-support)
- [uwsgi + paste](https://discuss.newrelic.com/t/uwsgi-paste-not-logging-data/34070)
- [uwsgi support paste](https://uwsgi-docs.readthedocs.io/en/latest/Python.html#paste-support)
- [RPC and nova](https://docs.openstack.org/nova/stein/reference/rpc.html)
- [oslo.messaging connection threads management (read, send)](https://github.com/openstack/oslo.messaging/blob/40c25c2bde6d2f5a756e7169060b7ce389caf174/oslo_messaging/_drivers/common.py#L369,L387)
- [openstack services and green threads (newton/OSP10)](https://docs.openstack.org/nova/newton/threading.html)
- [paste deploy project](https://github.com/cdent/paste/issues/25)
- [explainations from Chris Dent](https://github.com/cdent/paste/issues/25#issuecomment-502158572)
