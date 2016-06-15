---
layout: post
title: "Databricks cloud"
date: 2016-06-15 00:11:11
author: 伊布
categories: tech
tags: kerberos
cover:  "/assets/instacode.png"
---



### workspace

workspace有如下几部分：shared、user、folder。user下可以建folder、notebook、jar；folder下可以创建notebook。

workspace的权限控制

1、只有管理员可以打开、关闭权限控制功能。打开前已经存在的目录，仍然全局可见。

2、用户可以选择将notebook、folder授权给其他用户，权限包括：read、run、edit、manage。

![notebook permition](http://training.databricks.com/databricks_guide/2.8/ChangePermissions2.png)

![folder permition](http://training.databricks.com/databricks_guide/2.8/FolderPermissions2.png)

如果已经授权folder给某用户，那么该folder下所有item都会继承同样的权限。

3、libraries全局可见；share目录下内容全局可见。

workspce跟DB无关。一个workspace里的一个notebook，可以创建多个DB。


### UDF/library

1、用户可以添加自己的java/scala jar包到workspace里，然后在SQL notebook里create function xxx as 'org.dtdream.udf.Xxx'，就可以用这个UDF了。jar包从属于某个nodebook。集群重启后，jar、UDF仍然健在。
比较常用的做法。

2、用户可以将Python egg或者PyPI

3、用户可以将maven central或spark package里的库添加到library里。要求spark后面跟着maven库。


### cluster

用户登录到cloud上去以后可以创建自己的cluster。创建时，可以指定spark版本、指定excutor的形式（spot or on demand or hybrid）。

在一个workspace里可以有多个cluster；前面上传到space里的library，可以用到多个cluster。
当然我们可以一个workspace对应一个cluster先。

Databricks cloud里的cluster概念比较轻，不是我们平时理解的“集群”。


### 作业

用户通过studio，可以选择：

- 将某个notebook作为JOB提交到spark集群上
- 将spark应用的jar包丢到集群上并指定main class

可以设置JOB定期执行，设置JOB最长运行时间（超时则被关闭），可以设置邮件告警（如JOB启动、结束、出现错误时）。

作业提交到YARN集群（or mesos）上，天然的认证、隔离。


