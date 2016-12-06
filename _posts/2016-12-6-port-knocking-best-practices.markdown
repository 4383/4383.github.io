---
layout: post
title:  "Port knocking best practices"
date:   2016-12-6 12:55:28 +0100
categories: security, linux, port-knocking
---

# Port knocking best practices
In this post I want to explain how to use correctly port knocking with iptables.

## Prerequisite
- knowing SSH server management
- knowing Port knocking daemon management (knockd in this post)
- knowing iptables management

## Problem
When you setup your port knocking configuration every examples on internet explain to you to configure iptables firewall for reject all SSH traffic and to setup a rule in knockd configuration who allow SSH traffic in your iptables rules when the sequence is detected.

The problem, when you apply this kind of configuration any attackers with a port scanner like nmap can detect that the SSH daemon run on your server. Attacker know that your iptables have a filtering rule and know that SSH service run on port (by default 22).
```shell
# nmap -PN ww.xx.yy.zz

Starting Nmap 6.00 ( http://nmap.org ) at 2014-06-12 ...
Nmap scan report for xxxxxxxxxxxxx (ww.xx.yy.zz)
Host is up (0.0023s latency).
Not shown: 2043 closed ports
PORT     STATE    SERVICE
21/tcp   filtered ftp
22/tcp   filtered ssh
25/tcp   filtered smtp
80/tcp   open     http
```
This is a bad practice, you leak somes sensitives informations about your system configuration.
Anyone with a [tool for bruteforce port knocking like porno-king](https://mhackgyver-squad.github.io/porno-king/) can try to discover your port knocking sequence and open SSH service on this IP address.

## Solution
The right way to secure properly your server is:

1. by default you must shutdown your SSH service at start (or don't start at start)
2. configure iptables to reject all SSH traffic
3. setup knockd to start SSH service and add an entry to the iptables rules for allow connection on SSH service from IP address who have send the port knocking opening sequence

## Setup the best practices
### iptables setup
```shell
$ # use sudoers if necessary
$ iptables -A INPUT -i lo -j ACCEPT
$ iptables -F
$ iptables -X
$ iptables -A INPUT -j DROP
$ # now add your own iptables rules (http, smtp, ftp, etc...)
```
### knockd setup
knockd command to run on the opening sequence detection can be:
```shell
service ssh start && /sbin/iptables -A INPUT -s %IP% -p tcp --dport 22 -j ACCEPT
```
knockd command to run on the closing sequence detection can be:
```shell
service ssh stop && /sbin/iptables -D INPUT -s %IP% -p tcp --dport 22 -j ACCEPT
```
Or the entire configuration file (`/etc/knockd.conf`):
```shell
# /etc/knockd.conf
[options]
        UseSyslog

[openSSH]
        sequence    = 7000,8000,9000
        seq_timeout = 5
        command     = service ssh start && /sbin/iptables -A INPUT -s %IP% -p tcp --dport 22 -j ACCEPT
        tcpflags    = syn

[closeSSH]
        sequence    = 9000,8000,7000
        seq_timeout = 5
        command     = service ssh stop && /sbin/iptables -D INPUT -s %IP% -p tcp --dport 22 -j ACCEPT
        tcpflags    = syn
```

## Have Fun
Now try to scan your server port (SSH doesn't appear in the list of scanned service):
```shell
# nmap -PN ww.xx.yy.zz

Starting Nmap 6.00 ( http://nmap.org ) at 2014-06-12 ...
Nmap scan report for xxxxxxxxxxxxx (ww.xx.yy.zz)
Host is up (0.0023s latency).
Not shown: 2043 closed ports
PORT     STATE    SERVICE
21/tcp   filtered ftp
25/tcp   filtered smtp
80/tcp   open     http
```
