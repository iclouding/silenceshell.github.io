---
layout: post
title: "国内 docker 仓库镜像对比"
date: 2016-09-28 00:11:12
author: 伊布
categories: tech
tags: docker
cover:  "/assets/instacode.png"
---



首先，需要明确一个问题：Mirror 与 Private Registry 有什么区别？

抄自[daocloud的说明](http://docs.daocloud.io/faq/what-is-daocloud-accelerator)：

二者有着本质的差别。

Private Registry 是开发者或者企业自建的镜像存储库，通常用来保存企业内部的 Docker 镜像，用于内部开发流程和产品的发布、版本控制。
Mirror 是一种代理中转服务，我们(指daocloud)提供的 Mirror 服务，直接对接 Docker Hub 的官方 Registry。Docker Hub 上有数以十万计的各类 Docker 镜像。
在使用 Private Registry 时，需要在 Docker Pull 或 Dockerfile 中直接键入 Private Registry 的地址，通常这样会导致与 Private Registry 的绑定，缺乏灵活性。
使用 Mirror 服务，只需要在 Docker 守护进程（Daemon）的配置文件中加入 Mirror 参数，即可在全局范围内透明的访问官方的 Docker Hub，避免了对 Dockerfile 镜像引用来源的修改。

简单来说，Mirror类似CDN，本质是官方的cache；Private Registry类似私服，跟官方没什么关系。对我来说，由于我是要拖docker hub上的image，对应的是Mirror。
yum/apt的mirror又有点不一样，它其实是把官方的库文件整个拖到自己的服务器上做镜像（不管有没有用），并定时与官方做同步；而Docker mirror只会缓存**曾经**使用过的image。

目前国内访问docker hub非常便秘，使用docker mirror势在必行。现有提供服务的有三家：ustc、daocloud、aliyun，下面会一一介绍。

> 2017-03-06 更新：新增网易docker镜像

### 选择零：网易163 docker镜像

```
$ sudo echo "DOCKER_OPTS=\"\$DOCKER_OPTS --registry-mirror=http://hub-mirror.c.163.com\"" >> /etc/default/docker
$ service docker restart
```

一般来说，网易的猪肉比市场上的猪肉好吃。

### 选择一：ustc的镜像

ustc是老牌的linux镜像服务提供者了，还在遥远的ubuntu 5.04版本的时候就在用。之前在blog里有提到可以用ustc的docker仓库镜像，使用方法参考[ustc docker 镜像使用帮助](https://lug.ustc.edu.cn/wiki/mirrors/help/docker)。

ustc的docker镜像速度不错，一直用的挺happy。但是今天发现不好使了，可能跟[这件事](https://servers.blog.ustc.edu.cn/2016/09/mirrors-down-during-raid-rebuilding/)有关系吧，今天尝试去pull ubuntu，非常慢，应该是直接去docker hub上去拉了，基本没有加速效果。

ustc docker mirror的优势之一是，不需要注册，公共服务(这才是我熟悉的ustc嘛)。

### 选择二：daocloud

DaoCloud也提供了docker加速器，但是跟ustc不同，需要用户注册后才能使用，并且每月限制流量10GB。linux上使用比较简单，一条命令搞定：

```
curl -sSL https://get.daocloud.io/daotools/set_mirror.sh | sh -s http://{your_id}.m.daocloud.io
```

实际改的是/usr/lib/systemd/system/docker.service，加了个--registry-mirror参数，：

```
ExecStart=/usr/bin/docker-current daemon --registry-mirror=http://{your_id}.m.daocloud.io\
```

设置后，需要重新加载配置&重启：

```
systemctl enable docker; systemctl daemon-reload ; systemctl restart docker
```

但是！今天使用DaoCloud的docker加速器体验非常差，加速效果基本没感觉，果断放弃。

### 选择三：alicloud

阿里云也提供了docker加速器，不过比ustc更麻烦：不光要注册为阿里云的用户，还得加入开发者平台。不过捏着鼻子昨晚这些以后，它的服务还真是不错，基本1MB/s的pull速度(部分原因可能是因为我也在杭州吧)。配置方法跟daocloud类似，也是开通加速器以后给一个url。

我直接去改/usr/lib/systemd/system/docker.service了：

```
ExecStart=/usr/bin/docker-current daemon --registry-mirror=https://{your_id}.mirror.aliyuncs.com\
```

重新加载配置&重启：

```
systemctl enable docker; systemctl daemon-reload ; systemctl restart docker
```

pull的时候还是显示docker.io，但速度一点都不docker.io。

```
# docker pull ubuntu
Using default tag: latest
Trying to pull repository docker.io/library/ubuntu ...
latest: Pulling from docker.io/library/ubuntu
cad964aed91d: Pull complete
3a80a22fea63: Pull complete
50de990d7957: Pull complete
61e032b8f2cb: Pull complete
9f03ce1741bf: Pull complete
Digest: sha256:28d4c5234db8d5a634d5e621c363d900f8f241240ee0a6a978784c978fe9c737
Status: Downloaded newer image for docker.io/ubuntu:latest
```

----

综上，虽然aliyun docker mirror有这样那样的不爽，但的确是目前最好的服务。当然，等ustc恢复正常了，我还是会切到ustc的。
