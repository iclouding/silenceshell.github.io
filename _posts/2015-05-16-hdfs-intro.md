---
layout: post
title: "Hadoop文件系统：HDFS"
date: 2015-05-16 00:56:26
author: 伊布
categories: tech
tags: hadoop  hdfs
cover:  "/assets/instacode.png"
---


### 1 概述
HDFS应用场景：存储超大型流式处理数据（Terabytes和Petabytes级别）。
总的来说，HDFS的特点有这么几个：

- "write once, read many"，只支持一个writer，但对并发的reader支持很高的吞吐。
- 将数据处理逻辑放置到数据附近，可以减少数据拷贝的成本，提高并发效率。下面的读取/写入，都体现了这个特点。
- 可靠：维护同一文件的的多个副本+故障发生时自动重新部署问题节点。


从这几个不能满足的要求，可以反过来看HDFS的特点：

- 低延迟的数据访问。HDFS关注的是数据吞吐量，强调整个文件。
- 大量的小文件。HDFS设计为支持大文件，大量的小文件会造成namenode负载沉重。
- 多用户写入，任意修改文件。只支持一个writer，且只能写到文件的最后（流式处理）。
- 随机数据访问。

### 2 架构

- name node：管理文件系统命名空间和访问权限，记录了data node的数据块的位置（注意是在内存里，不保存，每次data node加入时重建）。
- data node：将数据作为块存储在文件里。

![](http://www.ibm.com/developerworks/cn/web/wa-introhdfs/fig1.gif)
HDFS架构如上图。name node可以识别data node的机架ID，从而优化data node之间的通信，减少跨机架的通信，因为通常这比同一机架的通信要慢。

#### 块
类似传统文件系统，HDFS也有**块**的概念，默认为64MB，但通常会被设置为128MB（跟硬盘读写速度相关）。数据存储到块文件里去。
块如此之大的原因：HDFS读取的吞吐率取决于2个方面：寻址的速度、块读取的速度。块设计的大一些，可以最小化寻址开销，尽可能接近硬盘的读取速度（极限情况：设计为只有一块，完全是硬盘的读取速度）。但不能将块设置太大，否则数据块的个数变少，会造成Map任务并发不够。
att：如果块存放的数据比块要小，实际并没有完全占用块。
使用块的优点：可以支持超大文件（经历过1080P拷贝到FAT32移动硬盘的同学会懂其中的痛）；相对管理文件来说，管理块更简单；可以做容错、负载分担。

#### 文件访问权限
如下例，hdfs跟POSIX非常像。hdfs的用户，实际就是远程客户端操作时候的用户，比如下面我用test上传了一个文件上去，其用户就是test，不管其他节点上是否也有test用户。hdfs的'x'权限，只是表示”该目录可进入“，没有”执行“这个概念。

```bash
[test@master conf]$ hadoop fs -put core-site.xml  /tmp/
[test@master conf]$ hadoop fs -ls /tmp/
Found 5 items
drwxrwxrwx   - hdfs   supergroup          0 2015-05-11 22:28 /tmp/.cloudera_health_monitoring_canary_files
-rw-r--r--   3 test   supergroup       3849 2015-05-14 19:10 /tmp/core-site.xml
..
```

新版本的hadoop使用了kerberos来做认证。

### 3 文件读取
#### 3.1 访问接口
hdfs提供了多种访问方式。
方式一：我们在客户端使用`hadoop fs -ls /`的操作，实际就是一个应用程序调用hdfs的java api接口实现的。
方式二：hdfs还提供了一个libhdfs的C语言库，可以通过JNI来访问；其他一些语言，例如c++, ruby, python等等可以通过[thrift](http://dongxicheng.org/search-engine/thrift-framework-intro/)代理来访问。thrift是个比较神奇的东西，后面可以好好玩一下。
方式三：hdfs提供了web浏览服务，下文提到的distcp就是利用了这一方式，其优势在于没有版本兼容的顾虑。

#### 3.2 客户端读取HDFS过程
![](http://upload.news.cecb2b.com/2014/1108/1415426527715.jpg?_=42872)
这个过程中，name node负责处理block位置的请求，客户端获得位置信息后，直接与data node联系，避免name node称为瓶颈。读取有下面两个特点

- 距离最近。读取datanode时，有一个“距离最近”的概念。name node返回存储该块的data node列表，dfs的datanode管理对象会从距离最近的datanode读取该数据块。该块read完毕后，继续请求下一块数据，直到读取完毕。
- 容错。datanode管理对象会校验读取数据，如果出错，会从其他最近的节点重新读取块，并且在读取下一block时通知name node。

那么距离是怎么定义的呢？可以将网络看成一棵树，两个节点的距离就是他们到最近的共同祖先的距离之和。根据数据中心、机架、节点，可以定义不同层级；对于复杂的网络，需要用户帮助hadoop来定义其拓扑。

### 4 文件写入
参考下面的代码：

```java
  org.apache.hadoop.fs.FileSystem hdfs = org.apache.hadoop.fs.FileSystem.get(config);
  org.apache.hadoop.fs.Path path = new org.apache.hadoop.fs.Path(filePath);
  org.apache.hadoop.fs.FSDataOutputStream outputStream = hdfs.create(path);
  outputStream.write(fileData, 0, fileData.length);
```

首先在name node文件系统的命名空间里创建一个新文件，注意这时还没有真正的数据块（实际就是一个记录）。
然后，数据被分为一个个数据包，一次性写入“数据队列”；数据队列的处理器会向name node请求得到一组data node作为一个“管道”；数据流式写入管道的第一个data node，然后再由这个节点同步给其他data node。
![](http://upload.news.cecb2b.com/2014/1108/1415426527902.jpg?_=6898)
只要有一个datanode写入成功就可以，集群会自行异步复制到其他节点。

> 问题：“管道”申请是每文件，还是每block？

name node是怎么选择data node来做副本写入呢？布局策略有很多，默认策略是这样的：先在写入数据的客户端本地存储一份（如果客户端不在HDFS集群上，则随机选择一个集群节点），然后在其他另一机架上选择2个节点各存储一份，再有副本就随机选择了。**只要写入一个data node成功，写操作就会成功，客户端调用只需要等待最小量的复制。**
该策略很好的平衡了可靠性（跨机架）、写入带宽（只需考虑本地交换机）、读取带宽（两个机架），数据均匀分布。

**数据一致性**
创建文件后，命名空间立即可见；但写入数据不能立即可见。应用程序需要自己权衡写入吞吐量和鲁棒，自行决定什么时候sync同步缓存。另外，文件关闭`close()`隐含了sync方法，需要所有数据写入datanode才能确认返回。

这里再谈一下客户端提交数据的过程。考虑这样一个问题：客户端读取小的日志并写入hdfs，每读取一条调用write，是否就引起写入hdfs一次？实际上客户端会建立一个临时本地文件，写入数据会被重定向到这个临时文件，直到满足一个数据块的大小；之后关闭临时文件，才会真正将此数据提交到hdfs上。

###5 可靠
**data node心跳**
data node需要有间隔的向name node发送心跳，如果心跳丢失，name node将该data node设置为不可用，如果导致副本低于**复制因子**，还需要将该data node上的数据块复制到其他的data node上。

**数据块再平衡**
情景一：当某些data node上的空闲空间太小时，hdfs会自动移动。
情景二：新增data node时。可以考虑手动平衡`hadoop balance`。

**数据完整**
对hdfs上的文件会存储checksum到文件系统命名空间的隐藏文件中；客户端取到数据后会检验这个checksum是否正确。

**元数据同步**
我们需要先了解几个概念，否则下面要讲的就完全看不懂了。
EditLog：一个记录文件系统命名空间的变换，例如文件重命名，权限修改，文件创建，数据块定位等等的持久化的文件。修改玩EditLog后，客户端的调用才会返回。
fsImage：实际就是文件系统周期性的还原点，也是持久化的。在fsImage基础之上，回放EditLog，可以得到最新的文件系统。
数据块位置记录：跟上面两个不同，存放在内存。name node启动时根据data node的报告重建。

简单的来说，name node的文件系统命名空间，权限，文件与数据块的映射关系，存储在fsImage文件中；所有用户对hdfs的操作，存放在EditLog文件中。hdfs对这些元数据文件可以做多副本（如写到远端NFS）。

**secondary namenode**
与namenode不同，主要工作是将命名空间fsImage和EditLog融合，并记录融合后的image，防止EditLog过大。其EditLog的同步滞后于name node。
name node故障后的过程如下：

- 将NFS上的image拷贝到secondary namenode上，加载到内存
- 重放EditLog记录的操作
- 收集足够的data node报告，重新建立数据块的映射关系
- 离开safe mode，成为正式的name node。

这一过程在大集群中可能会到几十分钟。

**HA**
hadoop 2.x版本比较好的解决了name node的单点故障。
新版本引入了name node**对**，分别配置为master/standby。故障恢复需要的两个关键信息是这样处理的：
![](http://pic002.cnblogs.com/images/2012/402771/2012120917500498.png)

- EditLog存放在NFS一类的共享存储中。master故障后，standby会立即重放EditLog，快速恢复状态。standby仍然运行着secondary namenode，用于融合image和EditLog。
- datanode需要向master/standby同时报告，两个name node都有完整的数据块映射。

据说只要几十秒就可以完成倒换。

### 6 Fedoration
引入了”域“的概念，允许集群不止有一个name node，但不同的name node共享同一个data node集群。
Fedoration可以解决集群变大时，name node成为瓶颈的问题，但是name node仍然存在单点故障，应属于一个过渡方案。不过Fedoration引入域以后，意外的获得了多租户的特性。

### 7 小文件解决方案
**归档工具**
archive可以克服大量小文件给namenode带来的内存压力，并且还能获得文件的透明访问。archive实际也是一个mr任务，需要hadoop集群。
[hdfs@master conf]$  hadoop archive -archiveName aaa.har -p /user/hdfs /user/hdfs
归档过程中是可以对文件进行压缩的，但是归档文件本身是不能压缩的。归档文件创建后不可更改，如果要删除或修改其中文件，只能重新归档。
**Sequence file**
由一系列KV组成，key为文件名，value为文件内容，将大批kv组成一个大文件。
这两种方案都需要用户应用程序干预，hdfs不会干预。

### 8 distcp
distcp是一个分布式并行的拷贝工具，实质是一个只有map的MR任务。通过制定map任务数(-m)，可以并行的将源文件拷贝到目标文件中。
相同版本的hdfs，distcp可以直接RPC拷贝；不同版本，则只能在目标hadoop上，通过http协议访问源dfs来并行的读取写入。
*数据平衡性*：如果distcp -m 1，当文件很大的时候，由于只有一个map任务，那么写入的时候总是会写入该map任务所在的节点，这就带来不均衡，所以尽量选择更多的map任务。

最后回答前面的问题：是每block的，因为hdfs的目的就是为了存储大文件，如果是每文件，就只能固定存在3个节点上了。


参考：
Hadoop权威指南
[IBM Developerworks](http://www.ibm.com/developerworks/cn/web/wa-introhdfs/)
[CSDN社区](http://www.cnblogs.com/beanmoon/archive/2012/12/11/2809315.html)
