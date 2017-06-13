---
layout: post
title: "docker不同版本重装后无法启动的问题"
date: 2017-06-12 00:11:12
author: 伊布
categories: tech
tags: kernel
cover:  "/assets/instacode.png"
---

我们使用的操作系统是centos7.2，yum源是内部的，其中docker版本是1.10.3；后来我将yum源换成外部的以后，重装docker，新的docker版本是1.12.6，如此操作可能会出现docker进程无法启动的问题。`systemctl status docker -l`查看docker进程状态，发现是inactive (dead) 状态，日志里查看提示是因为已经有docker0这个接口了。

```
Jun 07 10:33:21 localhost.localdomain forward-journal[44282]: time="2017-06-07T10:33:21.147037233+08:00" level=fatal msg="Error starting daemon:
Error initializing network controller: Error creating default \"bridge\" network:
cannot create network docker0 (ccdc8c7664a86ef2869ae3138aae0496569db18f4ca19558f8fe8d1bad75a342):
conflicts with network 9b442c5f7f03704417ad3e1d510a1c3d684b35dc1f438fda08a2b0ba1af312e4 (docker0):
networks have same bridge name"
Jun 07 10:33:21 localhost.localdomain systemd[1]: Started Docker Application Container Engine.
```

尝试ifconfig docker0 down && brctl delif docker0，然后重新安装docker，不好使。

查了下，发现这是docker早期版本的一个bug：[issue23630](https://github.com/moby/moby/issues/23630)

workaroud:

```
rm -rf /var/lib/docker/network
```

之后再重装docker。

在Jun 17, 2016之后的版本不会有这个问题。具体修改[参见这里](https://github.com/docker/libnetwork/pull/1271/files)。
