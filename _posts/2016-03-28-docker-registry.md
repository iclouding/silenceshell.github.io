---
layout: post
title: "搭建私有docker registry"
date: 2016-3-29 09:21:00
author: 伊布
categories: tech
tags: docker
cover:  "/assets/instacode.png"
---

由于docker没有CDN加速，所以我们在国内使用docker的时候，会感觉速度非常慢；另一方面，对于一些企业级应用来说，将自家应用放到Docker Hub中，总归是不太放心的。因此，需要做一个docker私有仓库，自己当家做主。

Docker私有仓库是[docker/distribution](https://github.com/docker/distribution)这个项目实现的。实际使用时，registry是以docker 容器的方式运行的，所以我们第一步就是怎么获得容器的镜像。有三个办法。

1. docker registry在Docker Hub有现成的镜像，你可以pull下来使用，但Docker Hub的速度实在是太感人了。命令很简单：`docker run -d -p 5000:5000 --restart=always --name registry registry:2`
2. 从github上下载registry包，从里面的Dockerfile可以build出来最终镜像。
3. 最好的办法，是问问周围有没有人做过，直接将他的docker registry镜像保存后自己load。

我选用的是第二个方法，没有为什么，练练手。选用第一个方法会比较简单。
另外，docker registry有v1和v2两个版本，我这里都是用的v2；v1已经不维护了。

### 1 准备registry镜像

#### 1.1 下载registry包，解压并build

```
wget https://codeload.github.com/docker/distribution/zip/docker/1.10
mv 1.10 xx.zip
unzip xx.zip  #得到distribution-docker-1.10目录
cd distribution-docker-1.10
docker build .
```

由于v2的registry使用的是go语言，所以需要先下载一个go docker镜像(相比较而言，v1使用python语言，需要下载一大堆包，v2轻巧一些)。下面是其中的Dockerfile，可以看到registry的base镜像是golang，需要安装几个apt包，之后设置GOPATH后make即可编译得到最终的registry可执行文件。

```
FROM golang:1.5.3

RUN apt-get update && \
    apt-get install -y librados-dev apache2-utils && \
    rm -rf /var/lib/apt/lists/*

ENV DISTRIBUTION_DIR /go/src/github.com/docker/distribution
ENV GOPATH $DISTRIBUTION_DIR/Godeps/_workspace:$GOPATH
ENV DOCKER_BUILDTAGS include_rados include_oss include_gcs

WORKDIR $DISTRIBUTION_DIR
COPY . $DISTRIBUTION_DIR
COPY cmd/registry/config-dev.yml /etc/docker/registry/config.yml
RUN make PREFIX=/go clean binaries

VOLUME ["/var/lib/registry"]
EXPOSE 5000
ENTRYPOINT ["registry"]
CMD ["/etc/docker/registry/config.yml"]

```

#### 1.2 启动registry容器

由于Dockerfile定义了ENTRYPOINT，所以在run的时候不需要再指定其要执行的程序。

```
# docker images
REPOSITORY                   TAG                 IMAGE ID            CREATED             SIZE
<none>                       <none>              35edc4804da9        5 seconds ago       825.7 MB
# netstat -antp|grep 5000
tcp6       0      0 :::5000                 :::*                    LISTEN      27263/docker-proxy
# docker run -d -p 5000:5000 35edc4804da9
# docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                    NAMES
5cab3904ae94        35edc4804da9        "registry /etc/docker"   2 seconds ago       Up 2 seconds        0.0.0.0:5000->5000/tcp   big_darwin

# docker exec -ti 5cab3904ae94 /bin/bash
root@5cab3904ae94:/go/src/github.com/docker/distribution#which registry
/go/bin/registry
```

进去后可以看到，registry是在/go目录下编译的。当然更好的做法是run的时候，-p指定为80:5000，这样在使用的时候，只要写地址即可（访问registry实际是http服务，默认80端口），会方便一些。

### 2 使用私有registry服务

#### 2.1 检查私有registry服务是否正常
使用curl命令检查registry能提供哪些repositories。

```
curl "http://192.168.103.88:5000/v2/_catalog"
{"repositories":[]}
```

显然是空的。

#### 2.2 上传镜像到私有registry
把本地的ubuntu:14.04镜像push到私有registry中。


```
# docker images
REPOSITORY                   TAG                 IMAGE ID            CREATED             SIZE
<none>                       <none>              35edc4804da9        20 minutes ago      825.7 MB
ubuntu                       14.04               2a956697b48a        24 hours ago        188 MB
# 打私有registry的标签
# docker tag ubuntu:14.04 192.168.103.88:5000/ubuntu:14.04
# docker images
REPOSITORY                   TAG                 IMAGE ID            CREATED             SIZE
<none>                       <none>              35edc4804da9        24 minutes ago      825.7 MB
192.168.103.88:5000/ubuntu   14.04               2a956697b48a        25 hours ago        188 MB
ubuntu                       14.04               2a956697b48a        25 hours ago        188 MB
# 上传
# docker push 192.168.103.88:5000/ubuntu:14.04
The push refers to a repository [192.168.103.88:5000/ubuntu]
1a649ccebd00: Pushed
5f70bf18a086: Pushed
1b82ce694c3b: Pushed
db6b2d84f3c6: Pushed
05b940eef08d: Pushed
14.04: digest: sha256:d0b93c80d9d46495195246ed4e3d6c34cbc4e0ba9580557024e40c57a49d3672 size: 1336

# curl "http://192.168.103.88:5000/v2/_catalog"
{"repositories":["ubuntu"]}

```

#### 2.3 使用私有registry

pull时加上私有registry的地址端口号即可。

```
docker pull 192.168.103.88:5000/ubuntu:14.04
```




