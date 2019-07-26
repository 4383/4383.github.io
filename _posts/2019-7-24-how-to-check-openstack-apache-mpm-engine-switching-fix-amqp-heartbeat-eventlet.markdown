---
layout: post
title:  "Compare apache MPM engine module on openstack"
description: "Comparison between the apache prefork module and event module, eventlet behaviour & performance benchmark"
date:   2019-7-24 16:31:00 +0100
image: apache.gif
categories: [openstack, apache, eventlet, greenlet, rabbitmq, heartbeat]
---
## Introduction

The goal of this post is to explain how to compare and test the behaviour and
performances introduced by switching the apache MPM engine from [`prefork`](https://httpd.apache.org/docs/2.4/fr/mod/prefork.html)
to [`event`](https://httpd.apache.org/docs/2.4/fr/mod/event.html).

The main idea is to check if using the apache MPM `event` module fix
the AMQP heartbeat issue previously described in my previous posts:
- [Apache MPM prefork - AMQP heartbeat and eventlet issue]({% post_url 2019-7-15-how-nova-consume-oslo-messaging %})
- [How to patch the apache engine in openstack]({% post_url 2019-7-23-how-to-patch-openstack-tripleo %})

These 2 previous links are useful to obtain the big picture about these changes.

## Who this article is for?

For developers and engineers who wants to observe the difference between using
an apache engine instead another (from [`prefork`](https://httpd.apache.org/docs/2.4/fr/mod/prefork.html)
to [`event`](https://httpd.apache.org/docs/2.4/fr/mod/event.html)).

For persons who facing issue by using eventlet green threads under an apache environment.

For persons who want to observe performances between apache engines.

For all our performance test we will use the
[Apache HTTP server benchmarking tool (`ab`)](https://httpd.apache.org/docs/2.4/programs/ab.html).

## Requirements

- An OSP15 already deployed
- An ssh access to your undercloud instance

## Unpatched environment

We first start by observing the current situation in a non patched environment.

## Reproduce the nova-api AMQP heartbeat issue

Connect you to your undercloud and become `stack` user and source the environment:

```shell
$ # you are there /home/stack
$ source stackrc
```

Check the current apache engine in use:

```shell
$ # list all the loaded modules
$ sudo podman exec -it nova_api apachectl -M
 authz_groupfile_module (shared)
 authz_host_module (shared)
 authz_owner_module (shared)
...
 dav_fs_module (shared)
 deflate_module (shared)
 dir_module (shared)
 env_module (shared)
 expires_module (shared)
 ext_filter_module (shared)
 filter_module (shared)
 include_module (shared)
 log_config_module (shared)
 logio_module (shared)
 mime_module (shared)
 mime_magic_module (shared)
 negotiation_module (shared)
 mpm_prefork_module (shared)
 rewrite_module (shared)
 setenvif_module (shared)
 socache_shmcb_module (shared)
 speling_module (shared)
...
 systemd_module (shared)
 unixd_module (shared)
 usertrack_module (shared)
 version_module (shared)
 vhost_alias_module (shared)
 wsgi_module (shared)
```

Then you can also `grep` the engine:

```shell
$ # check which engine is in use
$ sudo podman exec -it nova_api apachectl -M | grep -E "event|prefork"
 mpm_prefork_module (shared)
```

As you can see we've loaded the default apache engine (`prefork`).

Now try to reproduce the original issue in a non patched environment:

```shell
$ grep -ri /var/log/ -e "Unexpected error during heartbeart thread processing" -B 2 2>/dev/null
$ # no returned output
```

The previous command show to us that we don't triggered the error yet.

To trigger the error we need to call services like `nova-api`:

```shell
$ nova service-list 
+--------------------------------------+----------------+--------------------------+----------+---------+-------+----------------------------+-----------------+-------------+
| Id                                   | Binary         | Host                     | Zone     | Status  | State | Updated_at                 | Disabled Reason | Forced down |
+--------------------------------------+----------------+--------------------------+----------+---------+-------+----------------------------+-----------------+-------------+
| 0d8024fd-9605-446d-812d-2b7c500302dd | nova-conductor | undercloud-0.localdomain | internal | enabled | up    | 2019-07-25T08:48:53.000000 | -               | False       |
| e1127c3c-b49c-4a06-8783-03c0358b1256 | nova-scheduler | undercloud-0.localdomain | internal | enabled | up    | 2019-07-25T08:48:55.000000 | -               | False       |
| 4251e8c6-8874-403c-ad61-535daab1b464 | nova-compute   | undercloud-0.localdomain | nova     | enabled | up    | 2019-07-25T08:48:54.000000 | -               | False       |
+--------------------------------------+----------------+--------------------------+----------+---------+-------+----------------------------+-----------------+-------------+
$ grep -ri /var/log/ -e "Unexpected error during heartbeart thread processing" -B 2 2>/dev/null
/var/log/containers/nova/nova-api.log-2019-07-25 08:48:57.447 22 DEBUG nova.api.openstack.wsgi [req-9d369ea4-1812-4aee-b334-fdb0aa9ddd27 7a59ab2d44ff4d5ba9be756d6e1fe199 902e2a271f844f189dc61719ab44bc22 - default
default] Calling method '<bound method Versions.index of <nova.api.openstack.compute.versions.Versions object at 0x7f3bb2ed7ba8>>' _process_stack /usr/lib/python3.6/site-packages/nova/api/openstack/wsgi.py:523   
/var/log/containers/nova/nova-api.log-2019-07-25 08:48:57.448 22 INFO nova.api.openstack.requestlog [req-9d369ea4-1812-4aee-b334-fdb0aa9ddd27 7a59ab2d44ff4d5ba9be756d6e1fe199 902e2a271f844f189dc61719ab44bc22 - default default] 192.168.24.1 "OPTIONS /" status: 200 len: 405 microversion: - time: 0.001091
/var/log/containers/nova/nova-api.log:2019-07-25 08:48:58.731 19 WARNING oslo.messaging._drivers.impl_rabbit [-] Unexpected error during heartbeart thread processing, retrying...: ConnectionResetError: [Errno 104] Connection reset by peer
/var/log/containers/nova/nova-api.log-2019-07-25 08:48:59.211 19 INFO nova.api.openstack.requestlog [req-103ca1d1-36e5-40f5-bc02-ba5a27f91238 7a59ab2d44ff4d5ba9be756d6e1fe199 902e2a271f844f189dc61719ab44bc22 - default default] 192.168.24.1 "GET /v2.1" status: 302 len: None microversion: - time: 0.489079
/var/log/containers/nova/nova-api.log:2019-07-25 08:48:59.218 21 WARNING oslo.messaging._drivers.impl_rabbit [-] Unexpected error during heartbeart thread processing, retrying...: ConnectionResetError: [Errno 104] Connection reset by peer
/var/log/containers/nova/nova-api.log-2019-07-25 08:48:59.228 21 DEBUG nova.api.openstack.wsgi [req-ff7ec3ef-ffce-48a1-bb37-ff6f9bb88835 7a59ab2d44ff4d5ba9be756d6e1fe199 902e2a271f844f189dc61719ab44bc22 - default
default] Calling method '<bound method VersionsController.show of <nova.api.openstack.compute.versionsV21.VersionsController object at 0x7f3bb25899e8>>' _process_stack /usr/lib/python3.6/site-packages/nova/api/openstack/wsgi.py:523
/var/log/containers/nova/nova-api.log-2019-07-25 08:48:59.229 21 INFO nova.api.openstack.requestlog [req-ff7ec3ef-ffce-48a1-bb37-ff6f9bb88835 7a59ab2d44ff4d5ba9be756d6e1fe199 902e2a271f844f189dc61719ab44bc22 - default default] 192.168.24.1 "GET /v2.1/" status: 200 len: 388 microversion: 2.1 time: 0.012250
/var/log/containers/nova/nova-api.log:2019-07-25 08:48:59.338 22 WARNING oslo.messaging._drivers.impl_rabbit [-] Unexpected error during heartbeart thread processing, retrying...: ConnectionResetError: [Errno 104] Connection reset by peer
/var/log/containers/nova/nova-api.log:2019-07-25 08:48:59.339 22 WARNING oslo.messaging._drivers.impl_rabbit [-] Unexpected error during heartbeart thread processing, retrying...: ConnectionResetError: [Errno 104] Connection reset by peer
```

As you can see we now triggered the heartbeat issue by using the
`nova service-list` command.

Also we can take a look to the AMQP connection established between
`nova-api` and our `RabbitMQ` server:

```shell
$ sudo podman exec -it nova_api ss -4tnp | grep 5672 | wc -l
332
$ sudo podman exec -it nova_api ss -4tnp | grep 5672
...
ESTAB       0        0            192.168.24.1:3306        192.168.24.1:56722   
ESTAB       0        0            192.168.24.1:5672        192.168.24.1:60180   
ESTAB       0        0            192.168.24.1:5672        192.168.24.1:59500   
ESTAB       0        0            192.168.24.1:5672        192.168.24.1:33122   
ESTAB       0        0            192.168.24.1:5672        192.168.24.1:56362   
ESTAB       0        0            192.168.24.1:5672        192.168.24.1:60690   
ESTAB       0        0            192.168.24.1:5672        192.168.24.1:59498   
ESTAB       0        0            192.168.24.1:33246       192.168.24.1:5672    
ESTAB       0        0            192.168.24.1:59378       192.168.24.1:5672    
ESTAB       0        0            192.168.24.1:5672        192.168.24.1:56316   
ESTAB       0        0            192.168.24.1:59520       192.168.24.1:5672   
...
```

We can observe many connections opened for this service.

## Benchmark apache performances in a non patched environment

```shell
$ ab -n 1000 -c 10  https://192.168.24.2:13000/
This is ApacheBench, Version 2.3 <$Revision: 1843412 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking 192.168.24.2 (be patient)
Completed 100 requests
Completed 200 requests
Completed 300 requests
Completed 400 requests
Completed 500 requests
Completed 600 requests
Completed 700 requests
Completed 800 requests
Completed 900 requests
Completed 1000 requests
Finished 1000 requests


Server Software:        Apache
Server Hostname:        192.168.24.2
Server Port:            13000
SSL/TLS Protocol:       TLSv1.2,ECDHE-RSA-AES256-GCM-SHA384,2048,256
Server Temp Key:        X25519 253 bits

Document Path:          /
Document Length:        269 bytes

Concurrency Level:      10
Time taken for tests:   1.776 seconds
Complete requests:      1000
Failed requests:        0
Non-2xx responses:      1000
Total transferred:      555000 bytes
HTML transferred:       269000 bytes
Requests per second:    563.17 [#/sec] (mean)
Time per request:       17.757 [ms] (mean)
Time per request:       1.776 [ms] (mean, across all concurrent requests)
Transfer rate:          305.23 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        2    6   2.3      5      14
Processing:     4   12   3.1     12      24
Waiting:        4   12   3.1     11      24
Total:          6   18   3.9     17      32

Percentage of the requests served within a certain time (ms)
  50%     17
  66%     19
  75%     20
  80%     21
  90%     23
  95%     24
  98%     26
  99%     28
 100%     32 (longest request)
```

## Patch your env by using the apache MPM event module

Now you need to patch your environment to observe the behaviours changes
introduced by the apache MPM event module.

To patch your env you need to read my [previous article]({% post_url 2019-7-23-how-to-patch-openstack-tripleo %}).

This article describe [how to patch openstack with a different apache engine]({% post_url 2019-7-23-how-to-patch-openstack-tripleo %}).

## Try to reproduce the nova-api AMQP heartbeat issue with a patched env

Connect you to your undercloud and become `stack` user and source the environment:

```shell
$ # you are there /home/stack
$ source stackrc
```

Check the current apache engine in use:

```shell
$ # list all the loaded modules
$ sudo podman exec -it nova_api apachectl -M
...
 dav_fs_module (shared)
 deflate_module (shared)
 dir_module (shared)
 env_module (shared)
 expires_module (shared)
 ext_filter_module (shared)
 filter_module (shared)
 include_module (shared)
 log_config_module (shared)
 logio_module (shared)
 mime_module (shared)
 mime_magic_module (shared)
 negotiation_module (shared)
 mpm_event_module (shared)
 rewrite_module (shared)
 setenvif_module (shared)
 socache_shmcb_module (shared)
 speling_module (shared)
 ssl_module (shared)
 status_module (shared)
...
 usertrack_module (shared)
 version_module (shared)
 vhost_alias_module (shared)
 wsgi_module (shared)
```

Then you can also `grep` the engine:

```shell
$ # check which engine is in use
$ sudo podman exec -it nova_api apachectl -M | grep -E "event|prefork"
 mpm_event_module (shared)
```

As you can see we've loaded the default apache engine (`prefork`).

Now try to reproduce the original issue in a non patched environment:

```shell
$ grep -ri /var/log/ -e "Unexpected error during heartbeart thread processing" -B 2 2>/dev/null
$ # no returned output
```

The previous command show to us that we don't triggered the error yet.

To trigger the error we need to call services like `nova-api`:

```shell
$ nova service-list 
+--------------------------------------+----------------+--------------------------+----------+---------+-------+----------------------------+-----------------+-------------+
| Id                                   | Binary         | Host                     | Zone     | Status  | State | Updated_at                 | Disabled Reason | Forced down |
+--------------------------------------+----------------+--------------------------+----------+---------+-------+----------------------------+-----------------+-------------+
| 0d8024fd-9605-446d-812d-2b7c500302dd | nova-conductor | undercloud-0.localdomain | internal | enabled | up    | 2019-07-25T08:48:53.000000 | -               | False       |
| e1127c3c-b49c-4a06-8783-03c0358b1256 | nova-scheduler | undercloud-0.localdomain | internal | enabled | up    | 2019-07-25T08:48:55.000000 | -               | False       |
| 4251e8c6-8874-403c-ad61-535daab1b464 | nova-compute   | undercloud-0.localdomain | nova     | enabled | up    | 2019-07-25T08:48:54.000000 | -               | False       |
+--------------------------------------+----------------+--------------------------+----------+---------+-------+----------------------------+-----------------+-------------+
$ grep -ri /var/log/ -e "Unexpected error during heartbeart thread processing" -B 2 2>/dev/null
$ # no returned output
```

As you can see we don't `grep` any log error related to the AMQP heartbeat
during the last nova-api call. You only observe the previous traces recorded
in nova-api log file before we applied the patch.

Things seems works better now.

Also we can take a look to the AMQP connection established between
`nova-api` and our `RabbitMQ` server:

```shell
$ sudo podman exec -it ironic_api ss -4tnp | grep 5672 | wc -l
276
$ sudo podman exec -it nova_api ss -4tnp | grep 5672
...
ESTAB       0        0            192.168.24.1:3306        192.168.24.1:56722   
ESTAB       0        0            192.168.24.1:5672        192.168.24.1:60180   
ESTAB       0        0            192.168.24.1:5672        192.168.24.1:59500   
ESTAB       0        0            192.168.24.1:5672        192.168.24.1:33122   
ESTAB       0        0            192.168.24.1:5672        192.168.24.1:56362   
ESTAB       0        0            192.168.24.1:5672        192.168.24.1:60690   
ESTAB       0        0            192.168.24.1:5672        192.168.24.1:59498   
ESTAB       0        0            192.168.24.1:33246       192.168.24.1:5672    
ESTAB       0        0            192.168.24.1:59378       192.168.24.1:5672    
ESTAB       0        0            192.168.24.1:5672        192.168.24.1:56316   
ESTAB       0        0            192.168.24.1:59520       192.168.24.1:5672   
...
```

We can observe many connections opened for this service but we can less
connections than with the unpatched stack.

## Benchmark apache performances in a patched environment

```shell
$ ab -n 1000 -c 10  https://192.168.24.2:13000/
This is ApacheBench, Version 2.3 <$Revision: 1843412 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking 192.168.24.2 (be patient)
Completed 100 requests
Completed 200 requests
Completed 300 requests
Completed 400 requests
Completed 500 requests
Completed 600 requests
Completed 700 requests
Completed 800 requests
Completed 900 requests
Completed 1000 requests
Finished 1000 requests


Server Software:        Apache
Server Hostname:        192.168.24.2
Server Port:            13000
SSL/TLS Protocol:       TLSv1.2,ECDHE-RSA-AES256-GCM-SHA384,2048,256
Server Temp Key:        X25519 253 bits

Document Path:          /
Document Length:        269 bytes

Concurrency Level:      10
Time taken for tests:   1.881 seconds
Complete requests:      1000
Failed requests:        0
Non-2xx responses:      1000
Total transferred:      555000 bytes
HTML transferred:       269000 bytes
Requests per second:    531.55 [#/sec] (mean)
Time per request:       18.813 [ms] (mean)
Time per request:       1.881 [ms] (mean, across all concurrent requests)
Transfer rate:          288.10 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        2    6   2.6      6      17
Processing:     5   13   3.6     12      36
Waiting:        5   12   3.6     12      36
Total:          9   19   4.5     18      49

Percentage of the requests served within a certain time (ms)
  50%     18
  66%     20
  75%     21
  80%     22
  90%     24
  95%     26
  98%     29
  99%     34
 100%     49 (longest request)
```

We can observe some losed performances especially for requests per second:
- `531.55` for the patched version
- `563.17` for the non patched version

These results are not static, they can changes between benchmarks.

## Conclusion

We loose some performances but we gain more stability and fixing some vicious bugs.

Eventlet and green threads and async feature seems works better with the 
apache MPM event module.

[These changes are optional for the moment](https://review.opendev.org/#/c/671321/)
in openstack and not turned on.

The good news is that we don't need [`oslo.messaging` fixes](https://review.opendev.org/#/c/663074/)
to allow `nova-api` to use it properly in a monkey patched environment.

The both approaches (oslo.messaging fixes or apache engine switch) are optional
solutions, who need to be tested first in an isolated service by example, to measure
possible impacts and issues.
