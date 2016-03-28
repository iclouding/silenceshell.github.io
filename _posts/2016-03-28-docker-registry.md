---
layout: post
title: "搭建私有docker registry"
date: 2016-3-28 20:21:00
author: 伊布
categories: tech
tags: docker
cover:  "/assets/instacode.png"
---

由于docker没有CDN加速，所以我们在国内使用docker的时候，会感觉速度非常慢；另一方面，对于一些企业级应用来说，将自家应用放到Docker Hub中，总归是不太放心的。因此，需要做一个docker私有仓库，自己当家做主。

Docker私有仓库是[docker-registry](https://github.com/docker/docker-registry)这个项目实现的。实际使用时，registry是以docker 容器的方式运行的，所以我们第一步就是怎么获得容器的镜像。有三个办法。

1. docker registry在Docker Hub有现成的镜像，你可以pull下来使用，但Docker Hub的速度实在是太感人了。
2. 先下载ubuntu:14.04镜像，然后在这个基础上再从GitHub上下载docker registry，并使用其Dockerfile来build。当然，ubuntu的镜像稍小一些，不过速度也是挺感人的。
3. 最好的办法，是问问周围有没有人做过，直接将他的docker registry镜像保存后自己load。

我选用的是第二个方法，因为有天晚上我让机器跑了很久终于把ubuntu的base镜像pull下来了。

### 1 准备registry镜像
docker registry使用了ubunutu:14.04的基础镜像，在这上面做了哪些操作，我们可以先看docker-registry里Dockerfile文件。

```
# Latest Ubuntu LTS
FROM ubuntu:14.04

# Update
RUN apt-get update \
# Install pip
    && apt-get install -y \
        swig \
        python-pip \
# Install deps for backports.lzma (python2 requires it)
        python-dev \
        python-mysqldb \
        python-rsa \
        libssl-dev \
        liblzma-dev \
        libevent1-dev \
    && rm -rf /var/lib/apt/lists/*

COPY . /docker-registry
COPY ./config/boto.cfg /etc/boto.cfg

# Install core
RUN pip install /docker-registry/depends/docker-registry-core

# Install registry
RUN pip install file:///docker-registry#egg=docker-registry[bugsnag,newrelic,cors]
...
```
简单说一下这个Dockerfile.
1. 这个镜像基于ubuntu:14.04；
2. docker-registry是用python写的，所以需要python环境，以及pip（Python Package Index，类似apt、yum）。这里要下一大坨包，而默认ubuntu镜像的源是ubuntu官方的，很慢，所以下面我会把apt sourcelist改成163的源。
3. pip实际也是从他的默认仓库里来下载的，同样很慢，我这里改为了国内豆瓣的源。



#### 1.1 pull ubuntu 14.04镜像，修改其apt源为163，并commit
*凭记忆吧*

```
docker pull ubuntu:14.04
docker images
docker run -d -ti ${image ID} /bin/bash
docker ps
docker exec -ti ${container ID} /bin/bash
  #exec进入容器后修改/etc/apt/sources.list，把achives.ubuntu.com都改为mirrors.163.com，然后退出到宿主机
docker ps
docker commit ${container ID} ubuntu:14.04
```

注意要commit，不然docker build新registry镜像的时候还是用的base ubuntu。

#### 1.2 修改Dockerfile

如上所述，如果使用默认的pip仓库，速度会比较慢，甚至最终超时失败，所以需要改用国内的源，即在pip命令最后加上`-i http://pypi.douban.com/simple/`

```
# Install core
RUN pip install /docker-registry/depends/docker-registry-core -i  http://pypi.douban.com/simple/

# Install registry
RUN pip install file:///docker-registry#egg=docker-registry[bugsnag,newrelic,cors]  -i  http://pypi.douban.com/simple/
```

#### 1.3 编译镜像

直接编译即可。

```
cd docker-registry
docker build .
docker images
REPOSITORY                   TAG                 IMAGE ID            CREATED             SIZE
<none>                       <none>              efa9ef6e3948        17 minutes ago      430.5 MB
```

> todo...

docker run -d -p 5000:5000 -v /opt/docker/registry:/tmp/registry registry


root@Eevee:/opt/data/registry# netstat -antp|grep 5000
tcp6       0      0 :::5000                 :::*                    LISTEN      27263/docker-proxy

root@Eevee:/opt/data/registry# docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                    NAMES
b0dd3205a414        registry_80         "/bin/registry /etc/d"   2 weeks ago         Up 38 minutes       0.0.0.0:5000->5000/tcp   registry


docker exec -ti a6411f51e8df /bin/bash

root@a6411f51e8df:/# netstat -antp|grep 5000
tcp6       0      0 :::5000                 :::*                    LISTEN      - 




root@Eevee:~# curl "http://172.17.0.2:5000/v2/_catalog"
{"repositories":[]}


docker pull ubuntu:14.04
root@Eevee:~# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
ubuntu              14.04               97434d46f197        9 days ago          188 MB
ubuntu              latest              97434d46f197        9 days ago          188 MB
registry_80         latest              f6b7aaca4129        2 weeks ago         224.5 MB
hello-world         latest              690ed74de00f        5 months ago        960 B


root@Eevee:~# docker tag 97434d46f197 192.168.103.88:5000/ubuntu:14.04
root@Eevee:~# docker push  192.168.103.88:5000/ubuntu:14.04
The push refers to a repository [192.168.103.88:5000/ubuntu]
5f70bf18a086: Pushed 
1b82ce694c3b: Pushed 
db6b2d84f3c6: Pushed 
05b940eef08d: Pushed 
14.04: digest: sha256:9dfa2ec8293cf95aaf2b1d08e3afb590342ac8e844695bd7e334d07cbcf0d78d size: 4121



root@Eevee:/opt/docker/registry#  curl "http://172.17.0.2:5000/v2/_catalog"
{"repositories":["ubuntu"]}


http://blog.csdn.net/limingjian/article/details/40621233


