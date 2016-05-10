---
layout: post
title: "Spark（七）：Hive的授权"
date: 2016-05-19 08:11:11
author: 伊布
categories: tech
tags:
- hive
- spark
cover:  "/assets/instacode.png"
---

用户在使用Hive的时候，需要做到数据隔离，针对DB、表对不同的用户有不同的权限，即授权(Authorization)。注意跟身份验证(Authentication)区别，前者是判断用户对资源是否具有相应的权限，后者是验证用户的身份，即是否合法用户。这里先看Authorization。


### 使用场景

1. hive作为表存储层，hive提供表抽象和metadata存储，用户可以直接访问HDFS和metastore，我们的spark on hive即为此类。
2. hive作为查询引擎，最常用，具体又分为两类：

  a. hive命令行用户，即使用HIVE CLI，直接访问HDFS和metastore。现在基本不这么用了。
  b. JDBC/ODBC、Beeline，用户通过hiveserver2来访问HDFS和metastore，而不是直接访问。


### 三种授权模型


#### 1、Storage based Authorization

适合1和2a，用户直接访问HDFS和metastore。

- 对于HDFS，需要通过HDFS的授权模型，即目录、文件的rw来控制用户读写。
- 对于metastore，建议使能metastore的基于存储的认证（实际上我们直接在spark的hive-site.xml里明文填写了作为metastore的mysql的用户名、密码，这是很不安全的，试想下坏人登录到服务器上，拿到mysql的密码后进入数据库干点坏事，metastore就损坏了）。

不过我这里还没有配置metastore的授权。

#### 2、SQL standard based Authorization

Storaged bases authorization只能提供DB、表、分区级别的授权，对于列、view是无能为力的，需要提供SQL标准的授权（即我们熟悉的grant、revoke）。
适合2b。注意不能用于2a，因为hive cli模式下用户可以直接访问HDFS，可以bypass掉SQL授权的限制。
同样的，在我们目前spark on hive的模式下，thrift server可以直接访问HDFS（身份比较模糊，它基于hiveserver2，但其client type是HIVECLI而不是HIVESERVER2），hive jar中会检查client type，判断为CLI并且使能了HIVE_AUTHORIZATION_ENABLED会抛异常。所以无法配置SQL标准授权。

#### 3、默认授权模型

早起的hive版本提供了授权功能，可以grant、revoke进行权限操作，但是这个模型有个最大的漏洞：任意用户都具有grant权限！即使userA没有给userB授予DB_A的权限，userB也可以自己给自己grant DB_A的最高权限。官方的说法是为了防止“误操作”，但不能防止恶意攻击。
所以这个模型也不能使用。

### Storage based Authorization配置

上面三种授权方式，只能选择第一种。配置方法参考[官方说明](https://cwiki.apache.org/confluence/display/Hive/HCatalog+Authorization)。只需要配置2个：

```xml
    <property>
      <name>hive.security.authorization.enabled</name>
      <value>true</value>
    </property>

    <property>
    <name>hive.security.authorization.manager</name>
    <value>org.apache.hadoop.hive.ql.security.authorization.StorageBasedAuthorizationProvider</value>
    </property>

```

注意不要修改hive.server2.enable.doAs为false。doAs默认为true，即hiveserver2会以**调用**它的用户来执行hive操作，例如beeline登录时填写的用户。

验证：hdfs的/hive/warehouse目录是rwxr_xr_x，即只有用户dtdream有写权限，其他任意用户都只有读权限。下面我们用随意填写的xxx用户登录，可以读取dtdream用户的DB、表，但是不能创建表（DB级别无写权限），不能向表插入数据（表级别无写权限）。


```bash
hubt@spark1:/home/dtdream/spark/spark-1.6.1-bin-hadoop2.6/bin$ ./beeline
Beeline version 1.6.1 by Apache Hive
beeline> !connect jdbc:hive2://spark1:20000
Connecting to jdbc:hive2://spark1:20000
Enter username for jdbc:hive2://spark1:20000: xxx
Enter password for jdbc:hive2://spark1:20000: 
Connected to: Spark SQL (version 1.6.1)
Driver: Spark Project Core (version 1.6.1)
Transaction isolation: TRANSACTION_REPEATABLE_READ
0: jdbc:hive2://spark1:20000> use hive2db;
No rows selected (0.121 seconds)
0: jdbc:hive2://spark1:20000> select * from table1 limit 1;
+-----+-----------------+-----+------------+--+
| c1  |       c2        | c3  |     c4     |
+-----+-----------------+-----+------------+--+
| 1   | aa              | 1   | aa2        |
+-----+-----------------+-----+------------+--+
1 rows selected (0.393 seconds)
0: jdbc:hive2://spark1:20000> create table table5 (c1 int , c2 int);
Error: org.apache.spark.sql.execution.QueryExecutionException: FAILED: HiveException java.security.AccessControlException: Permission denied: user=xxx, access=WRITE, inode="/user/hive/warehouse/hive2db.db":dtdream:supergroup:drwxr-xr-x
	at org.apache.hadoop.hdfs.server.namenode.FSPermissionChecker.checkFsPermission(FSPermissionChecker.java:271)
	at org.apache.hadoop.hdfs.server.namenode.FSPermissionChecker.check(FSPermissionChecker.java:257)
...
0: jdbc:hive2://spark1:20000> insert into table1 select t.* from (select 101,"100.1.1.1", 33, "200.1.1.1") t;
Error: java.lang.RuntimeException: Cannot create staging directory 'hdfs://spark1/user/hive/warehouse/hive2db.db/table1/.hive-staging_hive_2016-05-10_11-24-45_162_8729653431604236433-3': Permission denied: user=xxx, access=WRITE, inode="/user/hive/warehouse/hive2db.db/table1":dtdream:supergroup:drwxr-xr-x
	at org.apache.hadoop.hdfs.server.namenode.FSPermissionChecker.checkFsPermission(FSPermissionChecker.java:271)
	at org.apache.hadoop.hdfs.server.namenode.FSPermissionChecker.check(FSPermissionChecker.java:257)
...

```


参考：
[hive官方资料](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+Authorization)




