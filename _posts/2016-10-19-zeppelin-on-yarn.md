---
layout: post
title: "Zeppelin配置spark interpreter on yarn"
date: 2016-10-19 00:11:12
author: 伊布
categories: tech
tags: zeppelin
cover:  "/assets/instacode.png"
---

在[Spark（五）：在Zeppelin中分析IPv4地址的瓜分图](http://www.datastart.cn/tech/2016/05/06/spark-5-zeppelin.html)中讲解了zeppelin的使用，但是这里的spark是standalone方式跑的，而实际上我们的环境是spark on yarn的，还需要有所修改；另外由于新环境是基于hortonworks的HDP搭的，有所不同，这里简单记录下。

[zeppelin官网](https://zeppelin.apache.org/docs/0.6.2/interpreter/spark.html)只是说明要修改spark interpreter的master为yarn-client，但还有些其他问题。

Q1：使用root启动zeppelin，提示无hdfs写入权限
A：切到hdfs，进入/home/hdfs，解压zeppelin压缩包。HDP版本不同组件分角色，以HDFS为例，只有hdfs用户有权限，而root在HDFS这里就是个渣渣，没有权限。

Q2：提示${hdp.version} bad substitution

```
:/usr/hdp/${hdp.version}/hadoop/lib/hadoop-lzo-0.6.0.${hdp.version}.jar:/etc/hadoop/conf/secure: bad substitution
at org.apache.hadoop.util.Shell.runCommand(Shell.java:576)
at org.apache.hadoop.util.Shell.run(Shell.java:487)
at org.apache.hadoop.util.Shell$ShellCommandExecutor.execute(Shell.java:753)
```

A：原因是HDP版本要求spark-commit的时候，需要-Dhdp.version=2.4.2.0-258（[参见](http://stackoverflow.com/questions/32341709/bad-substitution-when-submitting-spark-job-to-yarn-cluster)）。我们的集群已经改了spark的配置，但还需要在zeppelin这里加一下spark-submit的参数。需要修改zeppelin-env.sh：

```
export SPARK_SUBMIT_OPTIONS="--driver-memory 1G --executor-memory 2G --driver-java-options -Dhdp.version=2.4.2.0-258"
```

Q3：提示__spark_conf__xxxx.zip文件不存在

```
Application application_1476788223705_0007 failed 2 times due to AM Container for appattempt_1476788223705_0007_000002 exited with exitCode: -1000
For more detailed output, check application tracking page:http://zlatan2.dtdream.com:8088/cluster/app/application_1476788223705_0007Then, click on links to logs of each attempt.
Diagnostics: File file:/tmp/spark-53df2ea5-09d2-48e0-97e9-f6bbf8127860/__spark_conf__9219701849171231591.zip does not exist
java.io.FileNotFoundException: File file:/tmp/spark-53df2ea5-09d2-48e0-97e9-f6bbf8127860/__spark_conf__9219701849171231591.zip does not exist
at org.apache.hadoop.fs.RawLocalFileSystem.deprecatedGetFileStatus(RawLocalFileSystem.java:609)
at org.apache.hadoop.fs.RawLocalFileSystem.getFileLinkStatusInternal(RawLocalFileSystem.java:822)
at org.apache.hadoop.fs.RawLocalFileSystem.getFileStatus(RawLocalFileSystem.java:599)
at org.apache.hadoop.fs.FilterFileSystem.getFileStatus(FilterFileSystem.java:421)
```

A：HADOOP_CONF_DIR未配置或配置错误。需要修改zeppelin-env.sh：

```
# Options read in YARN client mode
export HADOOP_CONF_DIR=/usr/hdp/current/hadoop-client/conf
# yarn-site.xml is located in configuration directory in HADOOP_CONF_DIR.
```

注意需要指定到conf目录。

---

另，zeppelin已经出孵化器，进入apache正式版本了（目前是0.6.2）。新版本还支持用户认证，亚克西。
