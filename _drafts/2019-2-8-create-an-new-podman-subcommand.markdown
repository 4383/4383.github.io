---
layout: post
title:  "How to hack on podman"
description: "Setup your environment to start to hack on podman"
date:   2019-2-6 15:55:00 +0100
image: podman.jpg
categories: [containers, linux, podman, isolate, environment]
---
## Introduction
[podman](https://github.com/containers/libpod) is the next generation of Linux container tools.

During this post I'll try to describe how to improve podman by adding a new subcommand.
Podman is compatible with docker so the future command need to existing on docker.

## Prerequisites

- having some knowledges with the go language is a plus.
- having some knowledges with podman standard usages.
- knowning how to contribute to podman([further readings]({% post_url 2019-2-6-how-to-hack-on-podman %})).

I personaly start from scratch with go language and podman both so if you have
some good idea then go ahead.

## How to create a new podman subcommand



The first give to you some interesting commands like how to build podman from scratch etc...

The second help you to propose changes and to prepare your git commits.

All along this post we will use the both to propose your first changes to the project.

## Setup your environment

### Install requirements

First we need to install podman and some tools/dependencies like `runc` and `conmon`:

```shell
sudo dnf install -y \
    podman \
    atomic-registries \
    btrfs-progs-devel \
    conmon \
    containernetworking-cni \
    device-mapper-devel \
    git \
    glib2-devel \
    glibc-devel \
    glibc-static \
    go \
    golang-github-cpuguy83-go-md2man \
    gpgme-devel \
    iptables \
    libassuan-devel \
    libgpg-error-devel \
    libseccomp-devel \
    libselinux-devel \
    make \
    ostree-devel \
    pkgconfig \
    runc \
    skopeo-containers
```

- `runc` is a CLI for running open containers.
- `conmon` is a the Open Container Initiative (OCI) container runtime monitor.

Check that the installed version of `runc` is new enough by using:
```shell
$ runc --version
runc --version
runc version 1.0.0-rc6+dev
commit: d164d9b08bf7fc96a931403507dd16bced11b865
spec: 1.0.1-dev
```

When I wrote these line your version of `runc` need to be equal or greater than `1.0.0`
else you need to [build your own](https://github.com/containers/libpod/blob/master/docs/tutorials/podman_tutorial.md#installing-runc).

You also need to check the installed version of golang, `1.10.x` or higher is required:
```shell
$ go version
go version go1.10.3 gccgo (GCC) 8.2.1 20181215 (Red Hat 8.2.1-6) linux/amd64
```

## Fork and clone libpod

Libpod sources are available [here](https://github.com/containers/libpod). You need a github account to fork it and in
a second time clone your fork locally.

Libpod is a golang project and the project need to be clone under the `$GOPATH` directory.

Go language introduce the [`get` command](https://github.com/golang/go/wiki/GOPATH#repository-integration-and-creating-go-gettable-projects) who allow you to clone your fork:
```shell
$ go get github.com/<you>/libpod
```

Your fork was clone under your `~/go/src/github.com/<you>/libpod`.

You can also configure your go environment variables:
```shell
export GOPATH=~/go
export PATH=$PATH:$GOPATH/bin
```

You can also add these previous commands to your `bashrc`.

Further informations [about the `$GOPATH`](https://github.com/golang/go/wiki/GOPATH).

For me the previous command produce the following result:
```shell
$ go get github.com/4383/libpod
$ ls -l ~/go/src/github.com/4383/libpod
total 368
-rwxrwxr-x.  1 herve herve 70587  6 févr. 16:49 API.md
-rw-rw-r--.  1 herve herve 72123  6 févr. 16:49 changelog.txt
drwxrwxr-x.  3 herve herve  4096  6 févr. 16:49 cmd
drwxrwxr-x.  2 herve herve  4096  6 févr. 16:49 cni
-rw-rw-r--.  1 herve herve  2969  6 févr. 16:49 code-of-conduct.md
-rw-rw-r--.  1 herve herve 12232  6 févr. 16:49 commands.md
drwxrwxr-x.  3 herve herve  4096  6 févr. 16:49 completions
drwxrwxr-x.  9 herve herve  4096  6 févr. 16:49 contrib
-rw-rw-r--.  1 herve herve 10920  6 févr. 16:49 CONTRIBUTING.md
...
```
### Configure your CNI

First go to your clone dir by using `cd $GOPATH/src/github.com/<you>/libpod`

Run the following command:
```shell
sudo cp cni/87-podman-bridge.conflist /etc/cni/net.d
```

For further informations about [CNI network configurations and podman](https://github.com/containers/libpod/blob/master/cni/README.md).

### Prepare your clone for development

Ensure to always be inside your clone directory else `cd $GOPATH/src/github.com/<you>/libpod`.

Now install [needed tools](https://github.com/containers/libpod/blob/master/install.md#build):

```shell
make install.tools
make help # display available commands
```

## Start to hack on libpod

Find an [existing issue to fix](https://github.com/containers/libpod/issues), propose a new feature or an improvement.

### Create your working branch

To be sure to have an up-to-date environment you need to add a remote project
to your clone who allow to update from the official repository:
```shell
$ git remote add upstream git@github.com:containers/libpod
$ # get last update
$ git fetch upstream
$ # create a new branch up-to-date from upstream project
$ git checkout -b your-development-branch upstream/master
```

### Make your changes

Just before to start your hack you need to know some simply project rules:
- Only solve a problem per patch (by pull request)
- Explain your changes to be understandale by everyone (commit message).

You really need to *[take a look to the contribution guide](https://github.com/4383/libpod/blob/master/CONTRIBUTING.md#contributing-to-libpod) before starting your changes*.
Hack and make your changes on libpod/podman

### Commit and propose your changes

Now your changes was done you can commit them and propose them to the upstream
project by proposing a new pull request.

Before commiting your changes you need to take a look to the [developer certificate of origin](https://developercertificate.org/).

Now you can [commit your changes and sign-off your commit](https://github.com/4383/libpod/blob/master/CONTRIBUTING.md#sign-your-prs).

To sign-off your commit you need to add `Signed-off-by: <Your Name> <your.mail@mail.com>` to your commit message:

```shell
$ git commit
```

Here is an example of a commit message on [one of my libpod pull request](https://github.com/containers/libpod/pull/2283):
```shell
$ git log
commit f6f5d8b8134dba75f8f18be528ec544af2648001 (HEAD -> improve-makefile, origin/improve-makefile)
Author: Hervé Beraud <hberaud@redhat.com>
Date:   Wed Feb 6 18:27:00 2019 +0100

    Generate make helping message dynamicaly.
    
    Generate make helping message dynamicaly by using
    python code snippet inside Makefile.
    
    All commented make targets will be added to the
    help message. To be added to the helping message
    comment need to start with '## '.
    
    These specials comments are detected by the python code.
    Python code generate the helping output from these results.
    
    Notice that this commit introduce a dependency with python (compatible python 2 and 3).
    
    Signed-off-by: Hervé Beraud <hberaud@redhat.com>
...
```

Please take the time to provide an useful commit message with the goal of
your changes and the purpose. A beautiful git message is really useful when some
months later maintainers need to fix an issue or inspect the code. The git message
is like clues on a murder scene who allow developers to understand who, why, what, etc...

Push your changes to your clone and submit your pull request:
```shell
git push origin your-development-branch
```

## Review process

Now your changes was proposed you will receive some comments on your pull request.

By example you can [observe mine](https://github.com/containers/libpod/pull/2283).

This one had help me to write this post and to discover the podman project workflow.

In a first time you will receive some comments from CI bots who check if you have sign-off
your commit and some other comments like that.

When reviewers ask changes to you, you need to update your patch and I can propose
to you to [squash your commits](https://davidwalsh.name/squash-commits-git) 
in only once commit to maintain a more properly git history
since one pull request is equal to one semantic changes or purpose.

Repeat the last sentence until reviewers ask for changes to you and while your patch are not merged.

Well! Now enjoy your podman hacking and help the team to improve this project :)

Cheers!
