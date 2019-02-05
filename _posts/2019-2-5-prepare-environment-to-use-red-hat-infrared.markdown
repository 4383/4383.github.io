---
layout: post
title:  "Setup environment to use Red Hat Infrared"
description: "Setup your environment to use it for deploy openstack with Red Hat Infrared"
date:   2019-2-5 10:39:00 +0100
image: datacenter.jpg
categories: [virtualization, linux, openstack, infrared, environment]
---
## Introduction
[InfraRed](https://infrared.readthedocs.io/en/stable/index.html) is a plugin based system that aims to provide an easy-to-use CLI for Ansible based projects.
It aims to leverage the power of Ansible in managing / deploying systems,
while providing an alternative, fully customized, CLI experience that can be used by anyone,
without prior Ansible knowledge.

The project originated from Red Hat OpenStack infrastructure team that looked for a solution to
provide an “easier” method for installing OpenStack from CLI but has since grown and can be
used for any Ansible based projects.

In this post I want to explain how to setup your environment to use Red Hat Infrared
to deploy Openstack on it.

## Prerequisites
- an environment with Red Hat Enterprise Linux 7
- an ssh access to connect on your environment
- root access in a second time to use infrared on it

## Setup your environment

Environment setup is really straightforward you just need to install
some libraries and to restart the libvirt service at end:

```
$ yum install -y libvirt libguestfs libguestfs-tools virt-install
$ service libvirt restart
```

## Setup your infrared client

Now your environment is ready for use you need to configure laptop
to use infrared.

I suppose your laptop use fedora 29.

First start by installing dependencies:
```
$ sudo dnf install git gcc libffi-devel openssl-devel
$ sudo dnf install python-virtualenv
$ sudo dnf install libselinux-python
```

For further reading you can follow the [official setup documentation](https://infrared.readthedocs.io/en/stable/setup.html)

Now installing infrared:

```
$ git clone https://github.com/redhat-openstack/infrared.git
$ cd infrared
$ virtualenv .venv && source .venv/bin/activate
$ pip install --upgrade pip
$ pip install --upgrade setuptools
$ pip install .
```

## Provision your environment

Now we will to provision machines on your environment by using infrared.

Ensure to have the virsh plugin added:
```
$ infrared plugin add plugins/virsh
$ # if already activated you will receive a warning
```

Execute provisioning by providing some parameters through the CLI:
```shell
$ infrared virsh \
    --host-address $HOST \
    --host-key $HOST_KEY \
    --topology-nodes "undercloud:1,controller:1,compute:1"
```

You can connect to your host and check for vm provisioned:
```
[root@env ~]# virsh list
 ID    Name                           Status
----------------------------------------------
 1     controller-0                   running
 2     undercloud-0                   running
 3     compute-0                      running
``` 

Now you can start to deploy your openstack by using tripleo.

First check the tripleo plugins are available:
```
$ infrared plugin add plugins/tripleo-undercloud
$ infrared plugin add plugins/tripleo-overcloud
```

Deploy your undercloud by using OSP 10 (`newton`):
```
$ infrared tripleo-undercloud --version 10 --images-task rpm
```

Deploy your overcloud by using OSP 10 (`newton`):
```
$ infrared tripleo-overcloud \
    --deployment-files virt \
    --version 10 \
    --introspect yes \
    --tagging yes \
    --deploy yes
```

Normaly your setup is complet and you can start to use your openstack by creating newtork, vm, etc...

That's all!
