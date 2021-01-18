---
layout: post
title:  "qdr (qpid dispatch router) AMQP message routing for newbies"
description: "First steps with qdr"
date:   2021-01-18 19:41:00 +0100
image: qpid.png
categories: [qpid, qdr, podman, AMQP]
---
## Introduction

The goal of this post is to compile a simple recipe to setup quickly a
qdr (qpid dispatch router) to route AMQP messages between python clients.

It will allow us to get a simple environment to play with qdr/qpid locally.

This project provide a `Dockerfile` that qdr will be built and runned inside
a container by using podman. It will allow us to not modify our
laptop directly and mostly it will allow us to save the state of our
running instances for futures usages.

Notice that this post is mostly a personal reminder and a way to store/backup
my used commands.

## Prerequisites

To run this recipe you need to install the following tools:
- podman
- git

## Setup the router

### Get qdr

Clone the official repository and move inside the created directory:

```sh
$ git clone git@github.com:apache/qpid-dispatch
$ cd qpid-dispatch
```

## Run qdr

First build the container image:

```sh
$ sudo podman build . -t qdr
```

Launch a new container based on the image built previously:

```sh
$ sudo podman run -it --name qdr -p 5672:5672 localhost/qdr
```

Notice that we will redirect all the localhost trafic on 5672 to
this container.


## Run examples

[qpid-proton](https://qpid.apache.org/proton/) is an AMQP messaging toolkit.
It can be used in the widest range of messaging applications, including
brokers, client libraries, routers, bridges, proxies, and more.

It provide examples that can be used to test our router.

### Get qpid-proton

```sh
$ git clone git@github.com:apache/qpid-proton
$ cd qpid-proton
```

### Build qpid-proton

Install the needed dependencies:

```sh
sudo yum install \
    gcc \
    gcc-c++ \
    make \
    cmake \
    libuuid-devel \
    openssl-devel \
    cyrus-sasl-devel \
    cyrus-sasl-plain \
    cyrus-sasl-md5 swig \
    python-devel \
    ruby-devel \
    rubygem-minitest \
    jsoncpp-devel
```

I'm a fedora user so I install all the corresponding packaging by using
`yum` but please refer to the official installation guide for further details
or different distros.

```sh
$ mkdir build
$ cd build
$ cmake .. -DCMAKE_INSTALL_PREFIX=/usr -DSYSINSTALL_BINDINGS=ON
$ make install
$ cd /usr/share/proton/examples/python
```

### Run a receiver

```sh
# From /usr/share/proton/examples/python
$ python simple_recv.py -a localhost:5672/examples -m 5
```

### Run a sender

```sh
# From /usr/share/proton/examples/python
python simple_send.py -a localhost:5672/examples -m 5
```

You can see logs when the sender submit messages to the receiver.
