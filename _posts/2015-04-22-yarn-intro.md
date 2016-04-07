---
layout: post
title: "yarn（一）：介绍"
date: 2015-04-22 13:25:29
tags:  yarn hadoop
author: 伊布
categories: tech
cover:  "/assets/instacode.png"
---


### 1 设计理念和基本架构

编程模型（API）、数据引擎(Map Task, Reduce Task)、运行时环境(Job Tracker--Task Trackers)
MRv2修改了运行时环境，将资源管理调度与作业控制分离，YARN就是分离出来的资源管理模块（具体应用程序的作业控制由ApplicationMaster负责）。

YARN源代码
*hadoop-2.6.0/bin/yarn.sh:*

```bash
elif [ "$COMMAND" = "resourcemanager" ] ; then
  CLASSPATH=${CLASSPATH}:$YARN_CONF_DIR/rm-config/log4j.properties
  CLASS='org.apache.hadoop.yarn.server.resourcemanager.ResourceManager'
```

YARN仍然为主从模型，Resource Manager负责对各个Node Manager上的资源进行统一管理调度。
*hadoop-2.6.0/sbin/start-yarn.sh:*

```bash
# start resourceManager
"$bin"/yarn-daemon.sh --config $YARN_CONF_DIR  start resourcemanager
# start nodeManager
"$bin"/yarn-daemons.sh --config $YARN_CONF_DIR  start nodemanager
```

#### 1.1基本组成

ResourceManager(Resource Scheduler+Applications Manager)、Application Master、Node Manager、container

用户提交任务--->RM创建Application Master--->向Resource Manager申请资源--->Node Manager返回container--->AM将Task分配给对应的container
ASM管理AM，AM管理Task。

![内部任务关系](/image/YARN_internal.PNG)

资源分配单位：容器 `Resource Container`(CPU 内存)。YARN的Container跟lxc的Container概念有点类似，但是两个层次的概念。
可以认为，YARN的Container是一个抽象的概念，表示能力；lxc的Container是物理可见的，cgroup可以限制真正使用的cpu、内存等资源。当然，YARN实际也使用了cgroup来限制CPU，但是内存限制采用了线程监控的方案，没有用cgroup。
原因：cgroup设置内存限制后，如果该进程超了，会被系统直接杀死；而在YARN使用时可能存在进程在某一时刻使用内存量突然变大的情况，使用cgroup不太合理。
关于YARN的Container可以参考董西成的博客：
[YARN中container的概念](http://dongxicheng.org/mapreduce-nextgen/understand-yarn-container-concept/)
[YARN的资源隔离实现方式](http://dongxicheng.org/mapreduce-nextgen/hadoop-jira-yarn-3/)




#### 1.2 通信协议
RPC。典型的C/S架构，Server主要由ResourceManager启动，jobclient/node manager/application master/admin发起连接。
`proto buffer`  **序列化框架**

#### 1.3 工作流程
短应用程序、长应用程序(Storm/HBase)

1. 提交应用程序(AM/运行参数等)
2. RM向NM申请container，用来启动AM
3. AM向RM的ASM注册
4. AM向RM申请资源
5. AM申请到资源后，向NM请求启动任务
6. NM准备TASK的运行环境等，启动TASK
7. 各TASK向AM汇报自己的状态
8. AM注销、关闭

![YARN工作流程](/image/YARN_workflow.PNG)
