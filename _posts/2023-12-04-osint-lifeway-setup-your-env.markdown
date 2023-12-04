---
layout: post
title:  "The OSINT way of life"
description: "How to setup your environment to conduct OSINT investigations"
date:   2023-12-04 23:00:00 +0100
image: osint.png
categories: [OSINT, linux, investigations, security, hardening]
---
## Introduction

You are surely there because you made some online researches about how to
setup an OSINT environment, hopefully this post offer you some guide and
help you reaching your goals.

I just start my journey in the OSINT world, and so I took these notes during
the reading of the excellent book of Michael Bazzell, named,
[OSINT techniques: resources for uncovering online information](https://inteltechniques.com/book1.html).

I bougth this book last week, and I should admit I already fall in love for it.
To many good informations are available at each corner of this book. I think
this book can be considered as kind of bible for OSINT investigation. If you
not already read this book, please go buying it without waiting.

This resource is really affordable, even for newcomers.

I'm not a security expert, I already some security background, and I was
a member of security club, where we played CTF and security scenarios.
Learning OSINT is really a good completion even for technical profiles like
me.

Learning OSINT will teach you how your personal information can be leaked
online, and how to better control your public identity. Even IT professional
are not the best at controlling their personal data.

So, my notes are inspired from Michael's book and from Michael his online
magazine, [unredactedmagazine](https://inteltechniques.com/magazine.html). I
also added some extra personal tricks that found useful.
I experimented some of them years ago in an [old project
dedicated to Linux server hardening](https://github.com/4383/fabric-debian/).
Those are complementary here.

Put this online magazine in your bookmarks, you won't regret it.

Now lets start describing the 5 steps you need to prepare your OSINT hosting
environment.

One last advice before starting, you should never run any OSINT investigation
directly from your host. Your investigations will be run in a dedicated
and isolated virtual machine. More details about these VMs are available in
Michael's book. This article only speak about the host.

## 1. Isolate you

The first step to become an OSINT investigator is to be able to  do not leave
personal informations or trace about yourself.

So you should own a second machine fully dedicated to your OSINT sessions. An
old computer, a recycled one, would be enough and would allow you to keep
your OSINT session out of any personal informations which could threaten your
identity.

## 2. Be portable - USB device

The second step is to quickly setup a fully isolated environment where you
can run your session. Depending on the context you may need different kind of
environment. By example, an investigating environment to run OSINT session, an
anonyme environment to communicate in a safe manner and where your identity is
protected, a development environment to code things, a security hacking
environment to run security tools, or again, a testing environment to check
virus. Not all these environment are based on the same technologies, so you
need a way to setup any of them in minutes and in merely manner.

Ventoy is the ideal solution. You can use it to store all the O.S images of
these environment, on an USB key by example, and then boot your laptop on this
media to run the distros you need.

You can install it directly from the latest github release:
[https://github.com/ventoy/Ventoy/releases](https://github.com/ventoy/Ventoy/releases)

Once your USB media is setup by using ventoy I'd recommend download the
following iso:

- the latest LTS ubuntu version [https://ubuntu.com/download/desktop](https://ubuntu.com/download/desktop)
- the latest version of tails [https://tails.net/install/download/](https://tails.net/install/download/)
- the latest version of kali linux [https://www.kali.org/get-kali/#kali-platforms](https://www.kali.org/get-kali/#kali-platforms)
- the latest version of microsoft Windows 10 [https://www.microsoft.com/software-download/windows10ISO](https://www.microsoft.com/software-download/windows10ISO)
- the latest version of microsoft Windows 11 [https://www.microsoft.com/software-download/windows11](https://www.microsoft.com/software-download/windows11)
- the latest version of pfsense [https://www.pfsense.org/download/](https://www.pfsense.org/download/)

Owning these images can also help us to repair live broken installation of
these operating system.

To go further, I'd recommend you to read the excellent article "The linux
lifestyle - USB Multi-Boot Options" from the "unredacted" magazine wrote by
Michael Bazzell. You can find this article here:
[https://inteltechniques.com/issues/004.pdf](https://inteltechniques.com/issues/004.pdf)

## 3. Setup your laptop

You should start your OSINT journey on a virgin environment, freshly
installed. I'd recommend to install either an ubuntu or a fedora environment
on your hosting machine. Unless you are patient we need something easy and
quick to install and we don't want to spend hours configuring a distros that
would be potentially fully erased and reinitialized a couple of weeks later,
this is why I'd recommend using mainstream distros instead expert distros like
arch linux or even debian, that sometimes could be tricky and could require
setup hours.

For that you just have to boot over your USB stick and select the distro you
want between those you copied on your Ventoy media previously.

## 4. Secure your fresh install

Once your O.S is installed you want to check if your base environment is free
from virus and you want to keep it clean from undesired files. To do that we
want to run the following commands.

Firstly, install `clamav` to manage virus:

```shell
$ # on fedora
$ dnf install clamav clamav-freshclam
$ # on ubuntu
$ dnf install clamav clamav-daemon
```

Then, lets configure `clamav`:

```shell
$ sudo systemctl stop clamav-freshclam
$ sudo freshclam
$ sudo systemctl start clamav-freshclam
```

And then, now, trigger an analyze:

```shell
clamscan -r -i / # without deletion during the scan
clamscan -r -i --remove=yes / # to delete infected file
```

We can now check, by using `rkhunter`, if rootkits or backdoors are or not
installed in our environment, you can install this software by running:

```
$ dnf install rkhunter
$ apt install rkhunter
```

And then launch the check:

```
$ sudo rkhunter --update
$ sudo rkhunter --check
```

You should run `clamav` and `rkhunter` weekly.

## 5. Clean your fresh install

Then now it is time to clean your fresly installed env by
using [bleachbit](https://www.bleachbit.org/download). 

First, before erasing files, we want to  to preview all the undesired files
that would be removed if we run a real clean:

```shell
$ sudo bleachbit --list | grep -E "[a-z0-9_\-]+\.[a-z0-9_\-]+" | grep -v system.free_disk_space | xargs bleachbit --preview
```

If you agree with the previous preview then you can now trigger the clean:

```shell
$ sudo bleachbit --list | grep -E "[a-z0-9_\-]+\.[a-z0-9_\-]+" | grep -v system.free_disk_space | xargs bleachbit --clean
```

## 6. Securely store your password

The last step to finish to setup your hosting machine is to ensure to store
your account password in secured manner. We can use KeePassXC to do that:

```shell
$ dnf install KeePassXC # on fedora
$ apt install KeePassXC # on ubuntu
```

Then store all your secrets by creating a new dedicated KeePassXC database or
by reusing an existing one.

I'd recommend taking time to update all the passwords that you start storing
in your database by using the generator provided by KeePassXC.
Please, when the service provider allow it, do not hesitate to use the highest
level of security in your password. Unfortunatelly, even today, many online
services still use low security level...

## 7. Hardening your password policy

One last advice. 
I'd also recommend to activate the two factor authentication method with all
the service that propose it to you. Even today, too many online services do
not yet propose 2FA, that's a shame, think about it...

## Conclusion

Now that your OSINT host is ready, then you can start designing your OSINT
virtual machine and then begin your investigating session. You will find more
details about OSINT virtual machines in Michael's book.
