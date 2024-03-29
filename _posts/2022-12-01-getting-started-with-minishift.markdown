---
layout: post
title:  "Getting Started with Minishift - Personal notes"
description: "Personal notes to setup my minishift environment"
date:   2022-12-01 16:00:00 +0100
image: minishift.png
categories: [minishift, openshift, containers]
---
## Introduction

The goal of this post is to compile a simple recipe to setup quickly a
minishift environment. I take my note during my first deployment.

Notice that this post is mostly a personal reminder and a way to store/backup
my used commands.

## The context

I ran the following commands on my hypervisor through a ssh connection made
from my laptop.

## Get minishift

Get the latest version of minishift (1.34.3 at the time I wrote these lines)

```sh
# wget https://github.com/minishift/minishift/releases/download/v1.34.3/minishift-1.34.3-linux-amd64.tgz
# tar xfvz minishift-1.34.3-linux-amd64.tgz
# ln -s /root/minishift-1.34.3-linux-amd64/minishift /usr/local/bin/minishift
```

## Start Minishift

Start your minishift instance by passing [quay.io](https://quay.io/) as the
registry to use.

The goal is to avoid the docker registry limitation. For further details
please see this github issue https://github.com/minishift/minishift/issues/3521

```sh
# minishift start --docker-opt "add-registry=quay.io"
```

Once your env is fully deployed minishift display this message:

```
Server Information ...
OpenShift server started.
The server is accessible via web console at:
    https://192.168.42.231:8443/console

...
```

You can access the openshift console by requesting https://192.168.42.231:8443/console

## Access openshift console locally

As I settled my openshift env on a dedicated hypervisor I have to forward
my http requests to access my openshift console.

```
ssh -L 8443:192.168.42.231:8443 my-hypervisor
```

All the requests to 127.0.0.1:8443 will be forwarded to my hypervisor and
especially to 192.168.42.231:8443.

That's all, thanks for your reading.
