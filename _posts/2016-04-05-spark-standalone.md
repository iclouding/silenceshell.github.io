---
layout: post
title: "spark（二）：standalone集群部署"
date: 2016-04-05
author: 伊布
categories: tech
tags: spark
cover:  "/assets/instacode.png"
---


Spark有三种集群部署方式：

- standalone
- mesos
- yarn

其中standalone方式部署最为简单，下面做一下简单的记录。


### 1 环境
我在一台服务器上安装了ESXi来管理虚拟机，多个虚拟机组成spark集群。虚拟机软件配置如下：

- 安装ubuntu server 14.04版本
- 安装OpenJDK 1.7
- 用户名为dtdream，具备sudo权限，并且sudo执行命令时不需要输入密码
- master上配置所有节点的/etc/hosts记录
- 安装pssh

配置无密码sudo方法如下。
```
root@spark1:/etc# chmod u+w sudoers
root@spark1:/etc# echo "%dtdream ALL=(ALL) NOPASSWD: ALL" > sudoers
root@spark1:/etc# chmod u-w sudoers
```

我做的虚拟机镜像配置了/home/dtdream/.ssh/authorized_keys，默认打通了ssh免密登录。

### 2 部署
我们先部署1个master+3个slave的集群。感叹下，相比之下某阿公司的ADS产品，spark的部署真的太简单了。

我用的spark为目前的release 1.6.1 pre-build版本，官方下载for hadoop 2.6的即可。将该包丢到集群的各个机器上，解压。我解压后的路径是`/home/dtdream/spark/spark-1.6.1-bin-hadoop2.6`。

#### 2.1 单独启动
**启动master**
到spark1机器上，执行`./sbin/start-master.sh `，启动master。查看日志，可以找到如下2个路径：

- spark集群内master路径：`spark://spark1:7077`，下面slave启动时需要用这个路径来指定master
- WEB UI路径：`http://192.168.181.73:8080`，浏览器打开可以看到里面空空如也，什么资源也没有。

**启动slave**
到spark2机器上，执行`./sbin/start-slave.sh spark1:7077`，启动第一个slave，此时在WEB UI可以看到Alive Workers为1，core、memory为该slave节点的所有资源（内存扣1G给操作系统），workers中可以看到spark2。
继续到spark3、spark4机器上执行`./sbin/start-slave.sh spark1:7077`，最终在WEB UI上看到的情况如下：

![spark web ui](http://7xir15.com1.z0.glb.clouddn.com/spark1.JPG)

#### 2.2 集中启动
当然上面这样做很麻烦，需要在每个节点上启动进程，更好的办法是在master(spark1)上配置slaves节点，统一启动。

在spark1上：

```
cp ./conf/slaves.template ./conf/slaves
echo "spark2" >> ./conf/slaves
echo "spark3" >> ./conf/slaves
echo "spark4" >> ./conf/slaves
./sbin/start-all.sh
```

如果后面觉得slave资源不够了，仍然可以安装2.1的做法，在新节点上`./sbin/start-slave.sh spark1:7077`，可以动态的加载上去，并且在master上执行stop-all.sh，也可以把新加的这个节点stop掉（这么友好我都快哭了）

#### 3 验证
如上，资源已经有了，下面我们跑个简单的任务。

```
./bin/spark-submit --master spark://spark1:7077 --class org.apache.spark.examples.SparkLR --name SparkLR lib/spark-examples-1.6.1-hadoop2.6.0.jar
```

spark-submit可以指定master，我们这里因为是standalone，所以指定了spark1的路径，yarn/mesos不同。执行完毕后到WEB UI上可以看到该任务在3个slave节点上都跑了响应的作业。
还可以指定`--supervise`，如果应用返回非0值会做重试。

#### 4 资源调度
standalone模式下，任务遵从FIFO调度。由于默认任务会使用所有资源，所以同一时刻只能跑一个任务；不过我们可以限制每个任务占用的资源，这样可以多个用户同时使用。mesos或者yarn模式会更自由一些。

### HA

多个slave的情况下自动具备了worker的HA，因为spark会将失败的任务调度到其他worker上执行。但是，master还是有单点的，如果master故障了，那么用户就无法提交新的作业了。注意，已经提交到worker的作业不受影响。








[reference](http://spark.apache.org/docs/latest/spark-standalone.html)
