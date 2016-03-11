---
layout: post
title: "Greenplum试用"
date: 2016-03-11 11:08:02
author: 伊布
categories: tech
tags: greenplum
cover:  "/assets/instacode.png"
---

*搬运工又上班了！*

### 简介
Greenplum应用在OLAP领域，MPP架构，其底层使用Postgre，支持横向扩展，支持行存储、列存储，支持事务、ACID。
MPP数据库主打**share nothing**，即各节点间任何资源都不共享，从硬件的CPU/内存/网络/存储，到上层的操作系统，各节点都是独立的；节点间的交互主要通过网络进行通信。由于数据量越来越大，OLAP产品多采用MPP架构，例如阿里的ADS，百度的Palo。
相对于MPP，也有Oracle RAC这种**share everything**架构，各节点共享存储、内存客户互相访问。SMP由于只能使用一个节点，容易受到具体硬件的限制，但支持事务的效率比较高（不需要跨节点通信）；MPP扩展性好，更适应big data的场景，但多数MPP数据库不支持事务、ACID（例如阿里云ADS）。MPP数据库需要仔细设计数据的存储，避免出现大量的节点间shuffle通信，否则效率极低。
关于GP的架构分析，可以参考[这篇文章](http://www.cnblogs.com/daduxiong/archive/2010/10/13/1850411.html)。

Greenplum在2015年已经开源，Apache 2.0协议，源码可以访问[Github](https://github.com/greenplum-db/gpdb)。GP使用C语言，据说其代码非常漂亮。不过Greenplum的开源并不彻底，没有包含其新一代优化器orca（核心资产，咳咳）。不过从一些测试结果来看，开源版本与商业版本的性能差别不大，后面我们可能看到很多基于Greenplum包装的OLAP产品。



### 安装
Greenplum在其[官方网站](https://network.pivotal.io/products/pivotal-gpdb#/releases/669/file_groups/348)提供了用于RHEL(CentOS)、SuSE的版本。

GP的架构是典型的Master-Segments，主控节点和计算节点分离。

![High-Level Greenplum Database Architecture](https://raw.githubusercontent.com/greenplum-db/gpdb-sandbox-tutorials/gh-pages/images/highlevel_arch.jpg)

Master中只提供接入服务和元数据存储，用户实际数据存储在Segments中。GP使用UDP通信，但在UDP之上做了确认机制。没有使用TCP的原因是，TCP会限制最多1000个Segments（为什么呢？）。但从阿里实际使用的情况来看，UDP并不稳定，他们还是改用了TCP。

- 创建表时，需要指定distributed by（类似ADS的分区列，可以指定单列或多列，据此判断数据分布到哪个Segment）。如果不指定则使用第一列（吐槽下这个特性，建表这种频率很低的事情不应该以方便为主，而应该强制要求）。

```
create table faa.d_airports (airport_code text, airport_desc text) distributed  by (airport_code);
```

- 插入时，根据规则（如hash），不同主键的数据分配到不同的Segment存储。GP的hash算法可以做到数据基本均匀分布到各个Segment。GP也可以配置为数据随机分布到各个Segment。
- 查询时，由Master根据当前的Segments情况生成SQL执行计划，交给各个Segment执行；Master汇总各Segment执行结果，并返回给client

![Dispatching the Parallel Query Plan](https://raw.githubusercontent.com/greenplum-db/gpdb-sandbox-tutorials/gh-pages/images/dispatch.jpg)

但是GP的查询会发生shuffle。

- 数据装载时，使用gpfdist直接从外部数据源并行拖到Segments上，并且各Segments下载时并不是只下载自己的数据，而是先下载，发现不是自己的数据，再丢给正确的Segment。并行装载号称1小时2TB。

![External Tables Using Greenplum Parallel File Server (gpfdist)](https://raw.githubusercontent.com/greenplum-db/gpdb-sandbox-tutorials/gh-pages/images/ext_tables.jpg)


Segment支持横向扩展。

我的环境使用了3台虚拟机：1M+2S，虚拟机安装CentOS6.5，安装步骤参考[这篇文章](http://blog.csdn.net/gnail_oug/article/details/46945283)，非常详细。不过我这出过一个错误，免密码打通的时候，root没成功，后面手工配置的（比较奇怪，必须使用下面的命令来copy公钥，不能拷贝方式）。
```
ssh-copy-id -i .ssh/id_rsa.pub root@your_remote_host
```

### HA
GP支持为Master和Segment配置Mirror以提供高可靠性。

- Master->Mirror：使用postgre的日志同步功能。可以在部署时配置，也可以在Master运行时配置。
![Master Mirroring in Greenplum Database](http://gpdb.docs.pivotal.io/4330/graphics/standby_master.jpg)

- Segment->Mirror：如下图，一个Segment host可以配置多个Segment instance；以一个DB为例，配置mirror后，某一Segment instance会在另一Segment host上提供mirror instance。

![Segment Data Mirroring in Greenplum Database ](http://gpdb.docs.pivotal.io/4330/graphics/mirrorsegs.png)

相比GP，ADS的计算（类比Segment）可以2个实例同时工作，一定程度上可以提高查询性能。ADS管控（类比Master）也是单独的实例（主备），但以进程的形式存在，并不需要同步。GP的HA设计的不算太好，以Segment为例，默认HA策略是切换为Read-only，即只能读不能写；如果HA策略是continue，仍然可以读写，但当故障节点恢复后，最终需要重启GP集群（能忍？）。

更多详细HA信息，移步到[GP官网](http://gpdb.docs.pivotal.io/4330/admin_guide/managing/highavail.html)。

### 使用
安装完毕后，用户可以使用Postgre SQL的客户端来访问Greenplum。

### 查询详解

> todo..




