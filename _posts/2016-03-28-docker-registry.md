---
layout: post
title: "搭建私有docker registry"
date: 2016-3-28 20:21:00
author: 伊布
categories: tech
tags: docker
cover:  "/assets/instacode.png"
---

从这下载
https://github.com/docker/docker-registry

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


