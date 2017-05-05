---
layout: post
title: "杀死一只小鲸鱼"
date: 2017-05-05 00:11:12
author: 伊布
categories: tech
tags: docker
cover:  "/assets/instacode.png"
---

![docker banner](http://7xir15.com1.z0.glb.clouddn.com/docker-banner2.jpg)

之所以会有这个主题，是因为这几天我给mysql集群前置了一个keepalived，为了方便也做成了docker镜像，丢给k8s来部署。但实际测试时发现，当停止或者删除keepalived容器后，网卡上还残余之前keepalived下发的virtual ip。

直接在宿主机上安装keepalived不会有这个问题。从keepalived源码上来看，它会接管linux发给它的SIGTERM信号，之后清理现场，包括下发给接口的虚IP，所以问题原因就比较简单了，容器在退出时，并没用将SIGTERM信号传递给keepalived进程，造成keepalived强制退出，最终virtual ip残留。

要解决这个问题，还需要从容器退出开始谈起。先挖个坑，晚上填。


Ref:
[Gracefully Stopping Docker Containers](https://www.ctl.io/developers/blog/post/gracefully-stopping-docker-containers/)
[LVS + Keepalived](https://www.server-world.info/en/note?os=CentOS_7&p=lvs&f=2)

---
