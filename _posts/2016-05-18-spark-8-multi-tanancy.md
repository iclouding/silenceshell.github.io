---
layout: post
title: "Spark（八）：多租户隔离"
date: 2016-05-17 08:11:12
author: 伊布
categories: tech
tags: spark
cover:  "/assets/instacode.png"
---

### 1、SQL隔离

我们的SQL语句在studio中做保护，通过不同的workspace隔离。需要注意，由于workspace跟DB是一一对应的，useA将数据共享给userB的时候，userB只能在另一workspace，否则userB将看到userA的所有SQL。

### 2、UDF隔离

和SQL一样，通过studio的workspace隔离。

### 3、数据隔离

前文谈到，Spark 1.6.1的版本只能支持Storaged based authorization，即用户的数据隔离实际是通过HDFS的鉴权来实现。默认HDFS本身具备的用户权限管理基于linux，无法实现如下功能：

- 更细的粒度，HDFS只能到表，不能到列
- 多个用户组的权限管理，即多个用户只能从属于一个用户组，无法解决像两个部门不同权限的情况

需要引入一个工具，独立于HDFS/HIVE做统一鉴权，可选的有Ranger和Sentry。相比于Ranger，Sentry提供了HDFS sync功能，可以将用户在hive上SQL设置的权限设置同步到HDFS Acl（考虑这样的场景，用户在Hive上为某用户设置了只读权限，可以防止用户通过hive登录时写入，但如果用户通过MR、Spark等访问，就没办法限制了，因此需要再在HDFS对该用户设置对应的权限）。如果用Ranger，就只能设置两次，但如果是Sentry，通过HDFS sync可以自动的将hive的权限修改同步为HDFS设置权限，用户就不需要再在HDFS上操作一把了。

不过，由于我们用的是Spark提供的thrift-server，功能不全，无法实现Hive的SQL standard based authorization，用户无法用SQL做授权修改，所以目前来说用Sentry的意义不大，而且其设置需要通过HUE，相比之下Ranger直接提供了WEB UI，可能更方便一点，因此选择Ranger作为鉴权的工具。

考虑如下场景：用户yundun将hive2db数据库下的表table1授权给用户didi。

#### 3.1 用户登录

用户通过cas/uim登录到studio下不同的workspace，每个workspace对应一个DB，studio使用对应的用户到sts(spark thrift server)那创建DB。注意用户可以看到所有的DB（做不到这个级别的读隔离），但不能访问无权限的DB。

用户认证通过studio来做。studio收到用户读取数据的请求时，需要再将用户名、密码传递给sts，sts需要再做一遍认证。

#### 3.2 sts用户认证

有两个情况，一个上面说的studio访问sts，还有就是用户可能直接JDBC访问sts，这两种情况都需要sts具备用户认证的功能；而用户信息实际存储在uim里，我们有2个选择

- sts同步uim的信息，走自己的认证
- sts以插件机制检查用户认证

hiveserver2支持三种认证：kerberose、LDAP、自定义认证，对我们来说只能选择第三种，即sts将用户名和密码传递给自定义插件，由插件来认证（不抛异常就通过），sts不同步uim的用户信息。。所幸，sts用的hive-exec库里有自定义插件的interface（PasswdAuthenticationProvider）。

具体编码我会再写篇文章。

#### 3.1 用户同步

sts不存在用户管理的问题，只要将认证外包即可，但ranger不同，需要管理用户、组，认证外包解决不了用户管理的问题，因此ranger需要同步外部用户。有两个选择。

- Ranger有提供用户同步的功能，目前支持和linux用户、LDAP用户同步。可以考虑增加与UIM同步的功能。
- Ranger有提供用户操作的REST API，可以在cas中注册个钩子同步

Ranger目前支持与unix、LDAP、文件（csv/json等）做同步，以UNIX(linux)为例，我们在linux上useradd xxx，ranger-usersync进程会将该用户信息同步给ranger，我们在ranger这里就可以看到这个用户了，该用户属性为external。

但这里有个很致命的问题：外部同步的用户，其实没有同步密码，不能登录ranger系统，这样用户yundun就没办法登录到ranger上管理自己的数据，将表table1授权给用户didi了。所以只能选择在uim外挂插件，当uim有用户修改动作时，插件调用ranger的rest api，将用户同步到ranger系统中。


#### 3.2 授权

Ranger web ui作为跳转嵌入到studio中。
授权时，用户yundun跳转登录ranger系统（现在做不到sso，登录ranger需要再输一遍用户名密码），授权方式如下图。

![spark-ranger](http://7xir15.com1.z0.glb.clouddn.com/spark-ranger-1.png)

### 4 审计

使用Ranger的审计功能。

### 5 资源隔离

通过YARN做到CPU和内存的隔离（暂未实现）。

