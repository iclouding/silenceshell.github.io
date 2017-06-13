---
layout: post
title: "Databricks cloud初探"
date: 2016-06-15 00:11:11
author: 伊布
categories: tech
tags: databricks
cover:  "/assets/instacode.png"
---

* TOC
{:toc}

Databricks cloud（行文方便下面简称DC）是Databricks公司提供的公有云服务，用户在上面创建自己的spark集群，进行数据处理，其建设思路类似阿里云的ODPS。下面主要关注其权限控制。

### workspace

workspace有如下几个概念：user、folder、notebook、library。

- user：管理员可以邀请其他用户到自己的workspace里。通过workspace，可以控制notebook（类似studio工作流的节点）的权限。
- notebook：开发者主要呆在notebook中处理数据，例如%sql %scala %python等。跟Zeppeline不同，DC的notebook可以attach到不同的cluster，而Zeppelin只能通过配置Incepter连接到不同的spark集群，相对来说更生涩一点，不过功能基本上是一样的。
- library：例如用户自己开发的jar包、python egg等
- folder：可以将多个notebook、library放到一个folder里，方便管理。

workspace的权限控制的比较精细：

1、只有管理员可以打开、关闭权限控制功能。打开前已经存在的目录，仍然全局可见。

2、用户可以选择将notebook、folder授权给其他用户，权限包括：read、run、edit、manage。

![notebook permition](http://training.databricks.com/databricks_guide/2.8/ChangePermissions2.png)

![folder permition](http://training.databricks.com/databricks_guide/2.8/FolderPermissions2.png)

如果已经授权folder给某用户，那么该folder下所有item都会继承同样的权限，包括后面添加到该folder的item。

3、libraries全局可见。

4、workspace里有一个特殊的folder：share，里面的item全局可见。

workspce跟cluster、Database等是解耦的。一个workspace里的一个notebook，可以创建绑定不同的cluster；绑定后可以在该cluster里创建多个DB。


### UDF/library

全部web化，通过web上传。

1、用户可以添加自己的java/scala jar包到workspace里，然后在SQL notebook里create function xxx as 'org.dtdream.udf.Xxx'，就可以用这个UDF了。jar包从属于某个nodebook。集群重启后，jar、UDF仍然健在。
比较常用的做法。

2、用户可以将Python egg或者PyPI。PyPI的用法类似下面maven，Python egg还没用过。

3、用户可以将maven central或spark package里的库添加到library里。要求spark后面跟着maven库。


### cluster

用户登录到cloud上去以后可以创建自己的cluster。创建时，可以指定spark版本、指定excutor的形式（spot or on demand or hybrid）。

在一个workspace里可以有多个cluster；前面上传到space里的library，可以给多个cluster使用。

Workspace的管理员可以设置不同用户针对各cluster的[权限](https://docs.cloud.databricks.com/docs/latest/databricks_guide/index.html#02%20Product%20Overview/09%20Access%20Control/02%20Cluster%20ACLs.html)

当然对于我们来说，可能一个cluster打天下就够了。


### 作业

用户通过studio，可以选择：

- 将某个notebook作为JOB提交到spark集群上
- 将spark应用的jar包丢到集群上并指定main class

![upload a jar to run](http://training.databricks.com/databricks_guide/upload_job_jar.png)

![specify cluster](http://training.databricks.com/databricks_guide/2.8/jobs_conf_cluster.png)

可以指定是新建一个cluster，也可以指定在已经存在的cluster上运行；
可以指定worker的个数；
可以设置JOB定期执行，设置JOB最长运行时间（超时则被关闭），可以设置邮件告警（如JOB启动、结束、出现错误时）等等。

作业提交到YARN集群（or mesos）上，JOB间隔离。
在DC上JOB不能自己new sc，也不能关闭sc。这一点可以节省资源，但在多租户时会互相影响，可能每个JOB一个sc合适一些，

### JDBC

DC为JDBC开放了10000端口号，但默认只能在AWS内网里访问，如果用户BI需要使用需要结合AWS将使用公网IP，或者使用AWS的 Elastic IPAddress。
使用公网IP/主机名：

```
%scala
import scala.sys.process._
val hostname = "wget -O - http://169.254.169.254/latest/meta-data/public-hostname".!!

--2016-06-16 06:21:04--  http://169.254.169.254/latest/meta-data/public-hostname
Connecting to 169.254.169.254:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 48 [text/plain]
Saving to: ‘STDOUT’

     0K                                                       100% 3.82M=0s

2016-06-16 06:21:04 (3.82 MB/s) - written to stdout [48/48]

import scala.sys.process._
hostname: String = 
"ec2-52-88-888-88.us-west-2.compute.amazonaws.com
"
```

JDBC也需要用户认证，跟登录DC同一个用户。这要求Spark thrift server的认证，跟DC的认证要打通。在我们的系统里可以用CUSTOM读取UIM的方式做到。

---

总的来说DC把Spark工作空间的多租户做的比较细致，可以保护用户的知识产权，但并没有关注Cluster内DB的数据安全（全局可见）。在我们的系统里，这块还是要做的。看上去[Ranger plugin for Hive metastore](https://cwiki.apache.org/confluence/display/RANGER/Ranger+Plugin+for+Hive+MetaStore)能满足需求。

*慢慢更新中*