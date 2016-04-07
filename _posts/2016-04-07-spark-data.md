---
layout: post
title: "Spark（三）：数据导入"
date: 2016-04-07 08:11:11
author: 伊布
categories: tech
tags: spark
cover:  "/assets/instacode.png"
---

Spark提供了thrift server，可以提供HIVE2的JDBC连接。

启动方式：

```
./sbin/start-thriftserver.sh --master spark://spark1:7077
```

启动后，可以使用JDBC连接。我们用beenline简单测试下：


```
./bin/beenline
beeline> !connect jdbc:hive2://spark1:10000
0: jdbc:hive2://spark1:10000> create database xxx;
0: jdbc:hive2://spark1:10000> use xxx;
0: jdbc:hive2://spark1:10000> create table t1(c1 int , c2 int);
0: jdbc:hive2://spark1:10000> select * from t1;
+-----+-----+--+
| c1  | c2  |
+-----+-----+--+
+-----+-----+--+
No rows selected (0.147 seconds)
0: jdbc:hive2://spark1:10000> insert into t1 select t.* from (select 1,1) t;
0: jdbc:hive2://spark1:10000> select * from t1;
+-----+-----+--+
| c1  | c2  |
+-----+-----+--+
| 1   | 1   |
+-----+-----+--+
1 row selected (1.287 seconds)
```

很神奇的insert语句。