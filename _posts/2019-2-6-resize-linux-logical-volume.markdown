---
layout: post
title:  "Resize Linux Logical Volume LVM"
description: ""
date:   2019-2-6 12:29:00 +0100
image: datacenter.jpg
categories: [virtualization, linux, file system, logical, volume, environment]
---
## Introduction
Sometimes your machine is already settled at start and the standard
configuration doesn't reach to your needs.
For some particular reasons you would need to resize your mounting points.

By example, my goal is [to use infrared to deploy openstack on my personal hypervisor]({% post_url 2019-2-5-prepare-environment-to-use-red-hat-infrared %}).
Infrared create VMs and by default it use the root user to setup hypervor with
the virsh plugin. My hypervisor by default haven't enough space to create my topology (VMs) so
I need to resize my file system.

By default my hypervisor only have 50G availables for the `/` and 877G availables for the `/home`:
```shell
# df -h
File system                     Size    Used Available Use% Mounted on
/dev/mapper/my-lvm-root          50G      8G       42G  16% /
devtmpfs                         95G       0       95G   0% /dev
tmpfs                            95G       0       95G   0% /dev/shm
tmpfs                            95G     27M       95G   1% /run
tmpfs                            95G       0       95G   0% /sys/fs/cgroup
/dev/mapper/my-lvm-home         877G      0G        0G   0% /home
/dev/sda1                      1014M    147M      868M  15% /boot
tmpfs                            19G       0       19G   0% /run/user/0
```

My hypervisor is only for testing purpose. [Linux hardening](https://www.cyberciti.biz/tips/linux-security.html) is not
necessary on this environment and we don't need to apply security best practices.
This isn't a production environment.

We want to remove the `/home` and to reallocate available ressources to `/` so
at end `/` will have a size of 927G.

## Prerequisites
- ssh access to connect on your environment.
- root access or sudoers access.

## Resize your volumes

### Comment your fstab

First you need to remove `/home` from your `fstab` by commenting it.

Edit `/etc/fstab` and comment the line where `/home` appear.

### Unmount your /home

Now unmount your `/home`

```shell
$ umount /home
```

### Store volumes groups informations

```shell
$ VGINFO=$(vgs --noheading | awk '{print $1}')
```

### Stop the /home logical volume

By using the `lvchange` command you need to stop the `/home` logical volume:

```shell
$ lvchange -a n "${VGINFO}/home"
```

### Remove the /home logical volume

By using the `lvremove` command you need to remove the `/home` logical volume:

```shell
$ lvremove "${VGINFO}/home"
```

### Expand the root (`/`) logical volume and file system

Now by using the `lvresize` command you need to expand the `/` logical volume:

```shell
$ lvresize -f -r -l100%VG "${VGINFO}/root"
```

## Check your changes

You can verify that your changes was successfully applied by using the `df -h` command:

```shell
# df -h
File system                     Size    Used Available Use% Mounted on
/dev/mapper/my-lvm-root         927G      8G  917G   2% /
devtmpfs                         95G       0   95G   0% /dev
tmpfs                            95G       0   95G   0% /dev/shm
tmpfs                            95G     27M   95G   1% /run
tmpfs                            95G       0   95G   0% /sys/fs/cgroup
/dev/sda1                      1014M    147M  868M  15% /boot
tmpfs                            19G       0   19G   0% /run/user/0
```
