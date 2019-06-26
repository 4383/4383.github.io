---
layout: post
title:  "How to create and resize an instance on openstack newton/OSP10"
description: "Describe the commands needed to create a new instance and resize it on openstack newton"
date:   2019-6-26 15:34:00 +0100
image: openstack.jpg
categories: [openstack,OSP10,newton,instance,create]
---
## Introduction
The goal of this post is to show you how to create a new instance on a
openstack newton/OSP10 freshly deployed by using tripleO. Then when your
instance will be created we will try to resize it by using the `nova` commands.

In my case I've use infrared to deploy my openstack but the deployment isn't
the topic of this post, so feel free to deploy by using your own method or
if you want you can [follow my previous post who explain how to deploy openstack
by using infrared]({% post_url 2019-2-5-prepare-environment-to-use-red-hat-infrared %}) and [tripleO](https://wiki.openstack.org/wiki/TripleO).

## Prerequisites
- [openstack newton](https://www.openstack.org/software/newton/) deployed.

## Connection with your deployed openstack

First you need to connect you to your openstack plateform or [configure
your openstack client](https://docs.openstack.org/python-openstackclient/latest/configuration/index.html) to administrate your deloyed platform.

In this post we will be connected directly to your platform by using ssh and
avoid config steps in this post:
```shell
$ ssh <user>@<hypervisor>
```

You can observe your topology by using the `virsh` command:
```shell
[root@my-hypervisor ~]# virsh list
 ID    Name                           Status
---------------------------------------------
 4     undercloud-0                   running
 17    compute-0                      running
 18    compute-1                      running
 19    controller-0                   running
 20    controller-1                   running
 21    controller-2                   running
```

Now that you are connected to your hypervisor, then connect you to your
undercloud and become a `stack` user:
```shell
$ ssh root@undercloud-0
$ su - stack
$ source overcloudrc
```

## Launch an instance

### Configure your network

Create your self configured `selfservice` network:

```shell
[stack@undercloud-0 ~]$ openstack network create  --share --external \
>   --provider-physical-network provider \
>   --provider-network-type flat provider
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | UP                                   |
| availability_zone_hints   |                                      |
| availability_zones        |                                      |
| created_at                | 2019-06-26T15:55:10Z                 |
| description               |                                      |
| headers                   |                                      |
| id                        | be6aeec5-36a7-44bb-ba9c-c404f77323cc |
| ipv4_address_scope        | None                                 |
| ipv6_address_scope        | None                                 |
| mtu                       | 1500                                 |
| name                      | provider                             |
| project_id                | 5625a10a7c2b4156a0671183a362ccda     |
| project_id                | 5625a10a7c2b4156a0671183a362ccda     |
| provider:network_type     | flat                                 |
| provider:physical_network | provider                             |
| provider:segmentation_id  | None                                 |
| revision_number           | 3                                    |
| router:external           | External                             |
| shared                    | True                                 |
| status                    | ACTIVE                               |
| subnets                   |                                      |
| tags                      | []                                   |
| updated_at                | 2019-06-26T15:55:10Z                 |
+---------------------------+--------------------------------------+

```

Now create your subnet:

```shell
[stack@undercloud-0 ~]$ openstack subnet create --network provider \
>   --allocation-pool start=203.0.113.101,end=203.0.113.250 \
>   --dns-nameserver 8.8.4.4 --gateway 203.0.113.1 \
>   --subnet-range 203.0.113.0/24 provider
+-------------------+--------------------------------------+
| Field             | Value                                |
+-------------------+--------------------------------------+
| allocation_pools  | 203.0.113.101-203.0.113.250          |
| cidr              | 203.0.113.0/24                       |
| created_at        | 2019-06-26T15:57:08Z                 |
| description       |                                      |
| dns_nameservers   | 8.8.4.4                              |
| enable_dhcp       | True                                 |
| gateway_ip        | 203.0.113.1                          |
| headers           |                                      |
| host_routes       |                                      |
| id                | d57c2c4b-af08-4132-ab0f-3ffd66c2db0a |
| ip_version        | 4                                    |
| ipv6_address_mode | None                                 |
| ipv6_ra_mode      | None                                 |
| name              | provider                             |
| network_id        | be6aeec5-36a7-44bb-ba9c-c404f77323cc |
| project_id        | 5625a10a7c2b4156a0671183a362ccda     |
| project_id        | 5625a10a7c2b4156a0671183a362ccda     |
| revision_number   | 2                                    |
| service_types     | []                                   |
| subnetpool_id     | None                                 |
| updated_at        | 2019-06-26T15:57:08Z                 |
+-------------------+--------------------------------------+
```

### Create your flavor
```shell
[stack@undercloud-0 ~]$ openstack flavor create --id 0 --vcpus 1 \
  --ram 64 --disk 1 m1.nano
+----------------------------+---------+
| Field                      | Value   |
+----------------------------+---------+
| OS-FLV-DISABLED:disabled   | False   |
| OS-FLV-EXT-DATA:ephemeral  | 0       |
| disk                       | 1       |
| id                         | 0       |
| name                       | m1.nano |
| os-flavor-access:is_public | True    |
| properties                 |         |
| ram                        | 64      |
| rxtx_factor                | 1.0     |
| swap                       |         |
| vcpus                      | 1       |
+----------------------------+---------+
```

### Prepare your keypair

Use the following commands to create your own keypair:

```shell
[stack@undercloud-0 ~]$ ssh-keygen -q -N ""
[stack@undercloud-0 ~]$ openstack keypair create --public-key ~/.ssh/id_rsa.pub mykey
+-------------+-------------------------------------------------+
| Field       | Value                                           |
+-------------+-------------------------------------------------+
| fingerprint | ee:3d:2e:97:d4:e2:6a:54:6d:0d:ce:43:39:2c:ba:4d |
| name        | mykey                                           |
| user_id     | 58126687cbcc4888bfa9ab73a2256f27                |
+-------------+-------------------------------------------------+
[stack@undercloud-0 ~]$ openstack keypair list
+---------+-------------------------------------------------+
| Name    | Fingerprint                                     |
+---------+-------------------------------------------------+
| mykey   | ee:3d:2e:97:d4:e2:6a:54:6d:0d:ce:43:39:2c:ba:4d |
+---------+-------------------------------------------------+
```

### Configure your security rules

Now configure some security rules to allow [ICPM](https://docs.openstack.org/install-guide/common/glossary.html#term-internet-control-message-protocol-icmp)
ping and SSH access:
```shell
[stack@undercloud-0 ~]$ openstack security group list
+--------------------------------------+---------+------------------------+----------------------------------+
| ID                                   | Name    | Description            | Project                          |
+--------------------------------------+---------+------------------------+----------------------------------+
| 3ca0580b-13b9-4696-9278-15872585573a | default | Default security group | cb3899ad6a864221969909710665ed16 |
+--------------------------------------+---------+------------------------+----------------------------------+
[stack@undercloud-0 ~]$ openstack security group rule create --proto icmp default
+-------------------+--------------------------------------+
| Field             | Value                                |
+-------------------+--------------------------------------+
| created_at        | 2017-03-30T00:46:43Z                 |
| description       |                                      |
| direction         | ingress                              |
| ether_type        | IPv4                                 |
| id                | 1946be19-54ab-4056-90fb-4ba606f19e66 |
| name              | None                                 |
| port_range_max    | None                                 |
| port_range_min    | None                                 |
| project_id        | 3f714c72aed7442681cbfa895f4a68d3     |
| protocol          | icmp                                 |
| remote_group_id   | None                                 |
| remote_ip_prefix  | 0.0.0.0/0                            |
| revision_number   | 1                                    |
| security_group_id | 3ca0580b-13b9-4696-9278-15872585573a |
| updated_at        | 2017-03-30T00:46:43Z                 |
+-------------------+--------------------------------------+
[stack@undercloud-0 ~]$ openstack security group rule create --proto tcp --dst-port 22 default
+-------------------+--------------------------------------+
| Field             | Value                                |
+-------------------+--------------------------------------+
| created_at        | 2017-03-30T00:43:35Z                 |
| description       |                                      |
| direction         | ingress                              |
| ether_type        | IPv4                                 |
| id                | 42bc2388-ae1a-4208-919b-10cf0f92bc1c |
| name              | None                                 |
| port_range_max    | 22                                   |
| port_range_min    | 22                                   |
| project_id        | 3f714c72aed7442681cbfa895f4a68d3     |
| protocol          | tcp                                  |
| remote_group_id   | None                                 |
| remote_ip_prefix  | 0.0.0.0/0                            |
| revision_number   | 1                                    |
| security_group_id | 3ca0580b-13b9-4696-9278-15872585573a |
| updated_at        | 2017-03-30T00:43:35Z                 |
+-------------------+--------------------------------------+
```

### Create your instance

First observe the defaults available flavors:

```shell
[stack@undercloud-0 ~]$ openstack flavor list
+--------------------------------------+------------+-------+------+-----------+-------+-----------+
| ID                                   | Name       |   RAM | Disk | Ephemeral | VCPUs | Is Public |
+--------------------------------------+------------+-------+------+-----------+-------+-----------+
| 20995563-a42c-4ba2-b548-0dc827d597c7 | baremetal  |  4096 |    8 |         0 |     1 | True      |
| 69798aec-7d65-4101-9720-931ca33abda7 | compute    |  8192 |   47 |         0 |     1 | True      |
| fc90080b-57a5-4a3a-be97-c6e94be43de3 | controller | 20480 |   37 |         0 |     3 | True      |
+--------------------------------------+------------+-------+------+-----------+-------+-----------+
```
