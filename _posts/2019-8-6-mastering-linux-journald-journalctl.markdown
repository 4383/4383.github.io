---
layout: post
title:  "Mastering journald & journalctl"
description: "Learn how to use journald and journalctl"
date:   2019-8-6 11:53:00 +0100
image: journalctl.jpg
categories: [linux, journald, journalctl, debug, analyze]
---
## Introduction

The goal of this post is to explain how to use journald & journalctl and
their advanced features.

The main goal of this article is to store and share my useful commands and
knowledges about this tool.

## Requirements

- A GNU/Linux distros installed with journald & journalctl

## Read your logs with journalctl

### Boots

journalctl allow you to handle boots and to show related logs. This option
can be really useful to observe changes after a boot by example or to dig
on a specific period and related logs.

Read logs only since the latest boot with the `-b` option:

```shell
$ journalctl -b
```

Another interesting feature is the possibility to list all the system boots:

```shell
$ journalctl --list-boots
-61 4e097c8bf7304ed996f48dfb12b3fe1e Tue 2019-01-29 14:39:41 CET—Tue 2019-01-29 15:05:34 CET
-60 51dbdccb242f409da653fd23ced323fd Tue 2019-01-29 15:05:57 CET—Tue 2019-01-29 15:19:05 CET
-59 89b8aad887ff43b7b021058cdb6a3d91 Tue 2019-01-29 15:19:27 CET—Tue 2019-01-29 15:39:10 CET
...
-4 48b6446e309e4f2ca660794e95140a62 Thu 2019-06-27 18:04:03 CEST—Sun 2019-06-30 14:18:03 CEST
-3 a644a58083fe4458ab7e6809d742c57d Mon 2019-07-01 09:55:14 CEST—Tue 2019-07-09 16:39:27 CEST
-2 7445118158984765b44095865a030cc0 Tue 2019-07-09 16:39:41 CEST—Fri 2019-07-26 14:04:26 CEST
-1 2b0f7d47247046a3b23a5da18db59bae Fri 2019-07-26 14:04:49 CEST—Tue 2019-07-30 14:31:31 CEST
 0 f882c9662b3c4437be857ecabfd16be4 Tue 2019-07-30 14:31:57 CEST—Tue 2019-08-06 14:37:10 CEST
```

This output give you the boots order and the boots identifiers (IDs).

The order is sortered by dates where the last boot (the current boot) 
is represented by the `0` position (`f882c9662b3c4437be857ecabfd16be4`).

The most oldest boots are `-59, -60, -61`.

You can read logs from a specific boot by given the boot ID as a params
of the `-b` option:

```shell
$ journalctl -b 7445118158984765b44095865a030cc0
```

Or by given the corresponding index in the boot sorted order, where the order
will be determined by usage of the `-` (dash) in the index, examples:

```shell
$ journalctl -b -3 # will display the boot a644a58083fe4458ab7e6809d742c57d
$ journalctl -b a644a58083fe4458ab7e6809d742c57d # similar to the previous command
$ # then now reverse the order by removing the dash (-) from the index
$ # it will display the boot 89b8aad887ff43b7b021058cdb6a3d91
$ journalctl -b 3 
$ journalctl -b 89b8aad887ff43b7b021058cdb6a3d91 # similar to the previous command
```
