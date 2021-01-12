---
layout: post
title:  "Using podman to run Openstack oslo.messaging's simulator"
description: "Use podman to simulate RPC client server communication based on RabbitMQ with Openstack oslo.messaging"
date:   2020-12-4 11:33:00 +0100
image: broker.jpg
categories: [openstack, oslo.messaging, podman, rabbitmq]
---
## Introduction

[The oslo.messaging library](https://docs.openstack.org/oslo.messaging/latest)
is an openstack project that provide RPC mechanisms and notification.
The Oslo messaging library provides two independent APIs:

 - oslo.messaging.rpc for implementing client-server remote procedure calls
 - oslo.messaging.notify for emitting and handling event notifications

They are part of the same library for mostly historical reasons - the most
common transport mechanisms for oslo.messaging.notify are the transports used
by oslo.messaging.rpc.

Our goal in this post is to present how to setup quickly an environment to
allow to use oslo.messaging's simulator.

Indeed, oslo.messaging's simulator can be used to simulate various kind
of client/server. (notify/RPC), so it could be used to debug things on
oslo.messaging.

Here we will run our simulator against a RabbitMQ backend.

## Prerequisites

To run this recipe you need to install the following tools:
- git
- podman

## Setup your backend

First, start our RabbitMQ backend:

```sh
$ sudo podman run -d -e RABBITMQ_NODENAME=rabbitmq \
    -p 5672:5672 \
    -p 15672:15672 \
    --name rabbitmq rabbitmq:3-management
```

In the previous command we also forward the port `15672` to allow you to access
to RabbitMQ web dashboard. The dashboard can be accessed by visiting
`http://127.0.0.1:15672/#/` (username: guest, password: guest).

To setup my RabbitMQ server I used the official bitnami [RabbitMQ image](https://quay.io/repository/bitnami/rabbitmq).
You can retrieve her on [quay.io](https://quay.io/) or you search for many types of available images.

If everything works fine the previous command will output something like:

```
...
2020-12-04 10:55:26.621 [info] <0.724.0> Starting worker pool 'management_worker_pool' with 3 processes in it
2020-12-04 10:55:26.621 [info] <0.44.0> Application rabbitmq_management started on node rabbit@localhost
2020-12-04 10:55:26.621 [info] <0.549.0> Ready to start client connection listeners
2020-12-04 10:55:26.623 [info] <0.746.0> started TCP listener on [::]:5672
 completed with 3 plugins.
2020-12-04 10:55:26.774 [info] <0.549.0> Server startup complete; 3 plugins started.
 * rabbitmq_management
 * rabbitmq_web_dispatch
 * rabbitmq_management_agent
2020-12-04 10:55:26.774 [info] <0.549.0> Resetting node maintenance status
```

Now your RabbitMQ backend is ready to receive message and ready for the next
part of this recipe.

## Run oslo.messaging's simulator

### Prepare your configuration

Before starting our client/server we will define some minimal configuration
to allow us to activate/deactivate things more merely.

Here is the config sample that you need to put into a file locally:

```
# ~/tuto.conf
[DEFAULT]
# Activate the debug mode to get more verbose output and observe the
# messages content
debug = True
default_log_levels = ['oslo.messaging=DEBUG', 'amqp=DEBUG', 'oslo_messaging=DEBUG']
# Define the The network address for connecting to the messaging backend,
transport_url = rabbit://localhost/
```

### Run the RPC server

First, ensure that you have already cloned
[oslo.messaging's official repository](https://opendev.org/openstack/oslo.messaging)
and then move in the cloned directory.

```sh
$ git clone https://opendev.org/openstack/oslo.messaging.git
$ cd oslo.messaging
```

The simulator can be called directly by using `tox -e venv` that will generate
for you a fully setup virtual env and already fullfiled with all the requirements needed:

```sh
$ tox -e venv -- python ./tools/simulator.py -h
```

The previous command show you the simulator's help, some usages examples are given.

By example to instanciate a RPC server you can use:

```sh
$ tox -e venv --  python tools/simulator.py --config-file ~/tuto.conf rpc-server
```

Now move to [the connections tabs of the RabbitMQ' web dashboard](http://127.0.0.1:15672/#/connectionshttp://127.0.0.1:15672/#/connections)
and can observe that your simulator is now connected to your backend server.

![RabbitMQ active Connections](/images/blog/rabbit-connection-simulator.png)

### Run the RPC client

First, ensure that your RPC server is running (c.f the previous section).

Now run your client:

```
$ tox -e venv --  python tools/simulator.py --config-file ~/tuto.conf rpc-client
```

It should output something like:

```
...
venv run-test: commands[0] | python tools/simulator.py --config-file ~/tuto.conf  rpc-client
2020-12-04 14:18:12,907 INFO root Generating 1 random messages
2020-12-04 14:18:12,952 INFO root Messages has been prepared
2020-12-04 14:18:13,017 INFO root round-trip-0  : seq: 0    count: 0      bytes: 0         
2020-12-04 14:18:13,017 INFO root error-0       : seq: 0    count: 0      bytes: 0         
2020-12-04 14:18:13,017 INFO root client-0      : seq: 0    count: 1      bytes: 1675      
2020-12-04 14:18:14,019 INFO root round-trip-0  : seq: 1    count: 1      bytes: 1675       latency: 0.044     min: 0.044     max: 0.044    
2020-12-04 14:18:14,020 INFO root error-0       : seq: 1    count: 0      bytes: 0         
2020-12-04 14:18:14,020 INFO root client-0      : seq: 1    count: 0      bytes: 0         
2020-12-04 14:18:15,022 INFO root round-trip-0  : seq: 2    count: 0      bytes: 0         
2020-12-04 14:18:15,023 INFO root error-0       : seq: 2    count: 0      bytes: 0         
2020-12-04 14:18:15,023 INFO root client-0      : seq: 2    count: 0      bytes: 0         
2020-12-04 14:18:16,024 INFO root round-trip-0  : seq: 3    count: 0      bytes: 0         
2020-12-04 14:18:16,024 INFO root error-0       : seq: 3    count: 0      bytes: 0         
2020-12-04 14:18:16,025 INFO root client-0      : seq: 3    count: 0      bytes: 0         
2020-12-04 14:18:16,050 INFO root =================================== summary ===================================
2020-12-04 14:18:16,051 INFO root client: duration: 1.00 count: 1 (1.0 msg/sec) bytes: 1675 (1675 bps)
2020-12-04 14:18:16,051 INFO root error: duration: 0.00 count: 0 (0.0 msg/sec) bytes: 0 (0 bps)
2020-12-04 14:18:16,051 INFO root round-trip: duration: 1.00 count: 1 (1.0 msg/sec) bytes: 1675 (1675 bps) latency: 0.044 min: 0.044 max: 0.044
tools/simulator.py:774: DeprecationWarning: Using function/method 'get_transport()' is deprecated: use get_rpc_transport or get_notification_transport
  TRANSPORT = messaging.get_transport(cfg.CONF, url=args.url)
___________________________________ summary ____________________________________
  venv: commands succeeded
  congratulations :)
```

You can observe that a message have been successfully transmitted to your server.

Also you can observe [RabbitMQ's queues](http://127.0.0.1:15672/#/queues)
and also observe the reply queue used by the server to reply to the client.

## Simulator's Useful Features

### The Debug Mode

The simulator provide a debug mode that allow us to turn all logs at the
`DEBUG` level even for the underlaying libraries.

It is really useful to quickly track your application's debug messages.

It could by activated simply by passing the `-d` flag, example with a
rpc-server:

```
$ tox -e venv -- python tools/simulator.py --config-file ./tuto.conf -d rpc-server
```

### Wait Before Answer

In some situations you want to simulate failure, by example by removing a reply
queue to allow you to see what will happen. By example it could be used to
observe what will happen when a rpc server will send a reply on a undefined
reply queue.

But to do that you need first to remove manually an existing reply queue
declared by the rpc client. By removing the reply client the server will
reply on it and fails in some manner, and the client will reach a timeout by
waiting for the reply.


By using the `-w` param you can pass an integer and ask to the server to wait
for this duration before sending a reply. It will allow to manually remove the
corresponding queue.

Here is an example bellow.

Running a rpc server that will wait for the given duration
before replying to the client:

```
$ tox -e venv -- python tools/simulator.py --config-file ./tuto.conf -d rpc-server -w 40
```

Runnig a standard rpc client:

```
$ tox -e venv -- python tools/simulator.py --config-file ./tuto.conf -d rpc-client
```

Remove the reply queue when the client have send is message:

```
$ sudo podman exec -it rabbitmq rabbitmqctl delete_queue reply_<id>
```

Now observe the rpc server's logs.

A real life example can be found [here](https://bugs.launchpad.net/oslo.messaging/+bug/1905965/comments/8).

## Conclusion

Now you can start to play with oslo.messaging and observe how it works.
Don't hesistate to setup a cluster of backend server and to simulate scenarios
to simulate network partitions and observe how our RPC client/server react.
