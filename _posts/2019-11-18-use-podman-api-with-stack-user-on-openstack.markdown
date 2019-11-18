---
layout: post
title:  "Use the podman API on openstack with the stack user"
description: "Configure your environment to using the python-podman libraries"
date:   2019-11-18 18:53:00 +0100
image: podman.jpg
categories: [containers, python, podman, stack, openstack]
---
## Introduction
[podman](https://github.com/containers/libpod) is the next generation of Linux
container tools. You can run podman directly from the CLI by using the podman
commands (example: `podman ps`). For some reasons you can need to run podman
commands from python projects and instead of using some subprocess functions
like `subprocess.Popen` to trigger your commands you can use the
[python-podman](https://github.com/containers/python-podman) module.

[python-podman](https://github.com/containers/python-podman) is a python module
that allow you to execute podman commands against a podman host in your
python scripts/projects.

It's more preferable to chose this way to call podman from python because the
module ensure you to run compatible commands in version against your podman host.

`python-podman` is based on `varlink`. Varlink is used to create
[server/client](https://varlink.org/python/)
for podman where client could execute podman commands against a given host
through a dedicated socket.

Podman define a [varlink interface](https://github.com/containers/libpod/blob/e7540d0406c49b22de245246d16ebc6e1778df37/cmd/podman/varlink/io.podman.varlink), 
and then the `$ podman varlink` command will start a [varlink server](https://varlink.org/python/#varlink.Server) on your host and
create a socket that you can connect on with your client. Get more informations
about the [podman-varlink command](https://www.mankier.com/1/podman-varlink).

This varlink backend will allow your client to request the podman defined interface and
to handle (manage containers, images, etc...) your host (through the client server connection).

Openstack now use `podman` to manage containers. In openstack containers are
used to launch openstack services (nova, cinder, heat, etc...), and all sorts
of server and clusters (rabbitmq, mariadb, etc...).

The goal of this article is to show to you how to setup your openstack to 
allow your `stack` user to run podman commands by using the podman python API.
We don't want to use either the podman CLI, the sudo command or become root.

It can be a way to automatize things and that we can run as the `stack` user in
openstack.

## Prerequisites
You need an environment with the
[train release](https://www.openstack.org/software/train/) deployed. I suppose
you will deploy your openstack by using
[tripleo](https://docs.openstack.org/tripleo-docs/latest/) and by provisioning some
controllers/computes based on [RHEL8](https://developers.redhat.com/rhel8/)
to deploy your openstack.

## Check requirements

By using `ssh` connect you to your undercloud as the `stack` user:

```sh
$ ssh stack@<undercloud-ip>
```

Check if requirements are fully installed:

```sh
stack@undercloud ~$ rpm -qa | egrep -E "podman|varlink"
libvarlink-util-18-3.el8.x86_64
libvarlink-18-3.el8.x86_64
python-podman-api-1.2.0-0.1.gitd0a45fe.module+el8.1.0+4081+b29780af.noarch
podman-1.4.2-5.module+el8.1.0+4240+893c1ab8.x86_64
podman-manpages-1.4.2-5.module+el8.1.0+4240+893c1ab8.noarch
```

Here we can observe that `podman`, `libvarlink`, `python-podman` are installed.

Also we need to check if the `podman` group exist and if our `stack` user is
a member of this group:

```sh
stack@undercloud ~$ groups
stack podman
```

If the `podman` group doesn't exist create him:

```sh
stack@undercloud ~$ sudo groupadd podman
```

And then add the `stack` user to this group:

```sh
stack@undercloud ~$ sudo usermod -a -G podman stack
```

## Setup your environment

Starts the varlink service listening on uri that allows varlink clients to
interact with podman. If no uri is provided, a default URI will be used
depending on the user calling the varlink service.

The default for the root user is unix:/run/podman/io.podman.
Regular users will have a default uri of
[`$XDG_RUNTIME_DIR/podman/io.podman`](https://www.freedesktop.org/software/systemd/man/pam_systemd.html).

On undercloud all the containers are managed by the root.

You can do some tests with podman by using root or sudo to observe result:

```sh
stack@undercloud ~$ sudo podman ps --format "{{.ID}}"
dae04b68622c
60df83c4f676
b0ab3c3adf2d
stack@undercloud ~$ sudo su -
root@undercloud ~# podman ps --format "{{.ID}}"
dae04b68622c
60df83c4f676
b0ab3c3adf2d
```

As you can observe `stack` user is a sudoers.

On RHEL8 normally the socket is already setup during podman install
so you don't need to do something. You can check if the podman socket is
listening by using `systemctl`:

```sh
stack@undercloud ~$ sudo systemctl list-sockets
LISTEN                         UNIT                     ACTIVATES
/run/dbus/system_bus_socket    dbus.socket              dbus.service
...
/run/podman/io.podman          io.podman.socket         io.podman.service
```

If you see similar output then the socket is running and you can jump to
the section "Run podman commands by using the python API", else you need to
configure and start your backend.

The more simplest way to run your backend is to use the [`podman varlink`
command](https://www.mankier.com/1/podman-varlink).

Here we want to use the default socket setup during install and running our API
commands as the `root` user.

Run your backend on the default socket:

```sh
root@undercloud ~# podman varlink
```

Recheck normally now you will see your socket listening:

```sh
stack@undercloud ~$ sudo systemctl list-sockets
LISTEN                         UNIT                     ACTIVATES
/run/dbus/system_bus_socket    dbus.socket              dbus.service
...
/run/podman/io.podman          io.podman.socket         io.podman.service
```

Else you can try to follow some parts of this [blog post](https://podman.io/blogs/2019/01/16/podman-varlink.html)
and the following commands:

```sh
stack@undercloud ~$ sudo systemctl daemon-reload
stack@undercloud ~$ sudo systemd-tmpfiles --create
stack@undercloud ~$ sudo systemctl enable --now io.podman.socket
```

## Run podman commands by using the python API

At this stage normally your undercloud is setup and you can run podman
commands by using the podman/varlink API as the `stack` user.

You can try to get the list of all the running containers by using:

```sh
stack@undercloud ~$ varlink call -m unix:/run/podman/io.podman/io.podman.ListContainers
```

Normally you will obtain the same containers/output that if you runned the `podman ps`
command.

Now we will try to do the same things by creating a python
[varlink client](https://varlink.org/python/#varlink.Client):

```sh
stack@undercloud ~$ python -c 'import varlink; c=varlink.Client("unix:/run/podman/io.podman"); con=c.open("io.podman"); print(con.ListContainers())'
```

Under the hood the python-podman module do the same things that the previous command.

Now we can try to use the python-podman module directly:

```sh
stack@undercloud ~$ python -c 'import podman; c=podman.Client(); print("\n".join([el.id for el in c.containers.list()]))'
dae04b68622c
60df83c4f676
b0ab3c3adf2d
```

The previous example will give you the same output that if you have run:

```sh
stack@undercloud ~$ sudo podman ps --format "{{.ID}}"
dae04b68622c
60df83c4f676
b0ab3c3adf2d
stack@undercloud ~$ sudo su -
root@undercloud ~# podman ps --format "{{.ID}}"
dae04b68622c
60df83c4f676
b0ab3c3adf2d
```

As you can observe you can now call podman commands through the python API
by using the `stack` user.

You can start to implement your needs and your scripts based on this setup.

## Further readings

- http://www.projectatomic.io/blog/2018/05/podman-varlink/
- https://podman.io/blogs/2019/01/16/podman-varlink.html
- https://godoc.org/github.com/containers/libpod/libpod
- https://github.com/containers/libpod/blob/master/API.md
