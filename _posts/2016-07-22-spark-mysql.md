---
layout: post
title: "Spark通过JDBC使用MySQL作为数据源"
date: 2016-07-22 00:11:12
author: 伊布
categories: tech
tags: spark
cover:  "/assets/instacode.png"
---

假设场景：用户使用beeline或者其他JDBC客户端，通过Spark Thrift server的JDBC服务，来访问MySQL。一般来说，直接通过JDBC来访问MySQL，可以肯定其数据量不大（无论是从MySQL读还是写到MySQL），否则应该将MySQL的数据导入到Hive库中（当然可以使用create table xxx stored as parquet as select * from jdbc_table_X将数据写到hive表里，但体验并不好，基本等同于单线程）。

1.6.1版本可以参考官方的[说明](http://spark.apache.org/docs/1.6.1/sql-programming-guide.html#jdbc-to-other-databases)。注意官方说明还是以`SPARK_CLASSPATH=postgresql-9.3-1102-jdbc41.jar bin/spark-shell`这种方式来启动spark-shell的，但是1.6.1上会告警，最好还是用--jars指定。我使用的是thrift server提供的JDBC，启动命令是这样的：

```
./start-thriftserver.sh --master yarn-client --principal dtdream/zelda1@ZELDA.COM --keytab /etc/security/dtdream.zelda1.keytab --driver-java-options '-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=13838' --num-executors 9 --executor-cores 7 --executor-memory 30g  --driver-memory 30g --conf spark.yarn.executor.memoryOverhead=4096  --jars /home/dtdream/spark/spark-1.6.1-bin-hadoop2.6/lib/mysql-connector-java-5.1.35-bin.jar --driver-class-path /home/dtdream/spark/spark-1.6.1-bin-hadoop2.6/lib/mysql-connector-java-5.1.35-bin.jar:/usr/local/ranger-hive-plugin/lib/*:/usr/local/ranger-hive-plugin/lib/ranger-hive-plugin-impl/*
```

只使用--driver-class-path还不够，这个参数只会给driver指定jar包，但是Spark JDBC访问Mysql是发生在各个Excutor上的，还需要--jars为各个Excutor指定jar包，否则会报“no suitable driver found”。


**单分区表**


```
CREATE TEMPORARY TABLE jdbc_t1 USING org.apache.spark.sql.jdbc OPTIONS (url "jdbc:mysql://192.168.103.227/dxt?user=userName&password=userPasswd", dbtable "dxt.t1");
select count(*) from jdbc_t1;
create table xxx as select * from jdbc_t1;
```

由于指定了表只有1个分区，所以Select count的时候，只会起1个task。


**指定Partition**

```
CREATE TEMPORARY TABLE jdbc_copy USING org.apache.spark.sql.jdbc OPTIONS (url "jdbc:mysql://192.168.103.227/dxt?user=root&password=123456", dbtable "dxt.copy", partitionColumn "buer", lowerBound "1", upperBound "11", numPartitions "6");
select count(*) from jdbc_copy;
create table yyy stored as parquet as select * from jdbc_copy;
```

需要指定partitionColumn, lowerBound, upperBound, numPartitions，缺一不可。指定多个partition后，select count查询会起partition个TASK并发处理。从代码上来看，指定partition会在组织select语句的时候，带上whereClause，具体可以看JDBCRelation.columnPartition，但从实际体验上来看，貌似还是每个container全量抽取到内存了。


