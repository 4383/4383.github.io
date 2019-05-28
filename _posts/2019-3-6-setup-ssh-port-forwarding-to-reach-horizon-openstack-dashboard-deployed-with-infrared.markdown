---
layout: post
title:  "How to reach horizon with ssh forwarding for an Openstack deployed with Infrared and haproxy"
description: "Setup ssh hops join openstack horizon deployed by using Infrared and haproxy"
date:   2019-3-6 15:07:00 +0100
image: openstack-red-hat.jpg
categories: [virtualization, ssh, forwarding, horizon, openstack, infrared]
---
## Introduction
[InfraRed](https://infrared.readthedocs.io/en/stable/index.html) is a plugin
based system that aims to provide an easy-to-use CLI for Ansible based projects.
It aims to leverage the power of Ansible in managing / deploying systems,
while providing an alternative, fully customized, CLI experience that can be used by anyone,
without prior Ansible knowledge.

The project originated from Red Hat OpenStack infrastructure team that looked for a solution to
provide an “easier” method for installing OpenStack from CLI but has since grown and can be
used for any Ansible based projects.

In this post I want to explain how to setup ssh port forwarding to use an 
openstack horizon dashboard deployed by Infrared.

My post is inspired by [original post of Carlos Camacho](https://www.anstack.com/blog/2016/07/02/ssh-multi-hop-tripleo.html)
but since my topology differ from the original topology described in the Carlos post I want to write my own to keep
my actions saved and related to my needs and my topology. Thanks to Carlos for some given helps.

## Prerequisites
- an environment with Red Hat Enterprise Linux 7
- an ssh access to connect on your environment
- a fully deployed openstack environment (by using infrared).

## Connect to you hypervisor

```shell
$ ssh -L 38080:localhost:38080 root@<hypervisor-ip>
# login as root on your hypervisor
```

## Overview of my personal topology

```shell
$ virsh list
virsh list
 ID    Nom                            State
---------------------------------------------
 4     undercloud-0                   Running
 17    compute-1                      Running
 18    controller-0                   Running
 19    controller-1                   Running
 20    compute-0                      Running
 21    controller-2                   Running
```

## Connect to your undercloud

```shell
$ ssh -L 38080:localhost:38080 root@undercloud-0
# login as root on your undercloud
```

## Retrieve informations

```shell
# become stack user
su - stack
$ source stackrc
$ nova list
+--------------------------------------+--------------+--------+------------+-------------+------------------------+
| ID                                   | Name         | Status | Task State | Power State | Networks               |
+--------------------------------------+--------------+--------+------------+-------------+------------------------+
| d4cc3593-3490-4e8e-8515-7ad3d5aabc37 | compute-0    | ACTIVE | -          | Running     | ctlplane=192.168.24.18 |
| 371268c4-7917-4d0f-8bbb-87bfcd6c86a5 | compute-1    | ACTIVE | -          | Running     | ctlplane=192.168.24.19 |
| a7fadd7e-525e-4750-8e96-6bb4ac255005 | controller-0 | ACTIVE | -          | Running     | ctlplane=192.168.24.16 |
| f062bae1-1e7a-45f9-808b-7b0dd241d821 | controller-1 | ACTIVE | -          | Running     | ctlplane=192.168.24.13 |
| 4c3affcc-90a1-466b-83f4-483cf332a4d1 | controller-2 | ACTIVE | -          | Running     | ctlplane=192.168.24.6  |
+--------------------------------------+--------------+--------+------------+-------------+------------------------+
$ cat overcloudrc | grep OS_AUTH_URL
export OS_AUTH_URL=https://<your-controller-ip-to-forward-too>
$ cat overcloudrc | grep PASS
export OS_PASSWORD=<your-password-to-use>
```

## Connect to the horizon node

Now you just have to jump into the node where horizon live:

```shell
$ ssh -L 38080:<your-controller-ip-to-forward-too>:443 heat-admin@<your-controller-ip-to-forward-too>
```

## Open your browser

Now you just have to open [https://localhost:38080/dashboard/](https://localhost:38080/dashboard/) with your browser.

That's all!
