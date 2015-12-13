---
layout: post
title: "Spark（一）：介绍、初体验"
date: 2015-05-13 11:18:55
tags: spark
author: 伊布
categories: tech
cover:  "/assets/instacode.png"
---


###1 介绍
Spark是一个快速、通用的集群计算系统，提供JAVA/Scala/Python API，以及一系列的高级工具：Spark SQL/MLib/GrapyX/Spark Streaming.
Spark的编程语言是scala，同样采用scala的还有kafka。

###2 安装
我的环境使用的是CDH版本，安装时选上了Spark，手动安装请参考官网。
如果将Spark安装到hadoop上，需要注意其版本依赖关系。如果你的Spark底层使用了HDFS做存储，而Spark的版本与默认的hadoop一定要不同的话，需要自行编译，改脚本没用。

###3 核心：RDD
RDD(Resilient Distributed Datasets，弹性分布式数据集)模型的出现主要为了解决下面两个场景：

- 迭代计算
- 交互式数据挖掘：同一数据子集进行Ad-hoc计算

RDD解决了中间结果重用的问题，不再像MR模型必须写入HDFS。RDD具有下面两个特点：

- 分布式：分布在多个节点上，可以被并行处理；存储在HDFS或者RDD数据集，也可以缓存在内存中，从而被多个并行任务重用。
- 容错：某个节点挂掉后，丢失的RDD可以重构。

RDD支持两种操作：

- 转换。从现有RDD生成新的RDD，例如`map(func)`、`filter(func)`、`join()`等。
- 动作。将操作结果返回驱动程序或者写入存储，例如`reduce(func)`、`count()`、`saveAsTextFile()`

RDD还支持缓存，主要用在迭代计算中，转换时不用*再次计算*，用户可以用persist/cache等方法使中间结果的RDD数据集缓存在内存或磁盘中。
有关RDD的详细研究，可以参考CSDN的[这篇文章](http://blog.csdn.net/wwwxxdddx/article/details/45647761)。

###4 基本架构

Spark会将一个应用程序转为一组任务，分布在多个节点并行计算，并提供一个共享变量在这些并行任务间、任务与驱动程序间共享。
每个应用程序一套运行时环境，其生命周期如下：

- 准备运行时环境：资源管理。目前spark可以使用mesos（粗细两种粒度）/yarn（仅粗粒度）来作为其资源管理器。两种方式都需要BlockManager来管理RDD缓存。
- 将任务转换为DAG图。RDD父子数据集间存在宽依赖、窄依赖。连续多个窄依赖可以归并到一个阶段并行。
- 根据调度依赖关系执行DAG图。优化：数据本地性、推测执行。
- 销毁运行时环境

###5 Quick start
下面基本是参照[spark官网示例](http://spark.apache.org/docs/latest/quick-start.html)来的。
####5.1 spark shell
spark shell可以交互式的运行spark程序，可以查看中间运行结果，是个学习框架的好方法。

```
$ spark-shell
Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /___/ .__/\_,_/_/ /_/\_\   version 1.3.0
      /_/

Using Scala version 2.10.4 (Java HotSpot(TM) 64-Bit Server VM, Java 1.7.0_15)
...
scala>
```

我们可以在shell中进行一些scala的操作。shell启动时会自动提供一个SparkContext对象，我们可以直接用这个对象来加载文件为RDD。

```bash
scala> val textFile = sc.textFile("pi.py")		#textFile为一个RDD
textFile: org.apache.spark.rdd.RDD[String] = pi.py MapPartitionsRDD[3] at textFile at <console>:21
scala> textFile.count()		#RDD的action
res4: Long = 41
scala> textFile.filter(line => line.contains("Spark")).count()
res7: Long = 2				#RDD的transformation+action，先生成
15/05/12 19:00:25 INFO DAGScheduler: Job 2 finished: count at <console>:24, took 0.184386 s
```

spark也提供了一个python语言的shell，运行`pyspark`可以进去，使用起来跟scala类似，具体可以参见官网。

RDD提供了跨集群、内存级的*cache*功能，对于一些频繁访问的数据集生成缓存，可以提高效率。

```bash
scala> val linesWithSpark = textFile.filter(line => line.contains("Spark"))
scala> linesWithSpark.collect()
res12: Array[String] = Array(from pyspark import SparkContext, "    sc = SparkContext(appName="PythonPi")")
scala> linesWithSpark.cache()
res13: linesWithSpark.type = MapPartitionsRDD[5] at filter at <console>:23
scala> linesWithSpark.count()
res16: Long = 2
scala> linesWithSpark.count()
res17: Long = 2
```

####5.2 Self-Contained Applications
实际生产环境中不可能像上面这样交互式运行，还是要自包含的应用程序。下面我们举一个源代码里python的例子，scala和java需要配合sbt和maven，具体请参照官网。

```python
import sys
from random import random
from operator import add
from pyspark import SparkContext

if __name__ == "__main__":
    """
        Usage: pi [partitions]
    """
    sc = SparkContext(appName="PythonPi")			#sc要自己创建
    partitions = int(sys.argv[1]) if len(sys.argv) > 1 else 2
    n = 100000 * partitions

    def f(_):
        x = random() * 2 - 1
        y = random() * 2 - 1
        return 1 if x ** 2 + y ** 2 < 1 else 0

    count = sc.parallelize(xrange(1, n + 1), partitions).map(f).reduce(add)
    print "Pi is roughly %f" % (4.0 * count / n)

    sc.stop()	#停掉sc
```

简单来说，spark应用程序比shell方式多了SparkContext的创建、销毁，其他基本是一致的。
使用spark-submit将此python脚本提交到spark执行：

```bash
$ spark-submit pi.py 10
15/05/12 19:42:26 INFO DAGScheduler: Job 0 finished: reduce at /opt/cloudera/parcels/CDH-5.4.0-1.cdh5.4.0.p0.27/lib/spark/examples/lib/pi.py:38, took 1.867190 s
Pi is roughly 3.142040
```

如果python程序依赖其他的文件或第三方的lib库，可以将其打包为zip文件，用--py-files指定就可以了。python系统库不需要。

Spark提供了一个history web server，可以看到我们运行了的那些程序以及他们的详细信息。
![spark jobs](http://7xir15.com1.z0.glb.clouddn.com/spark_jobs.PNG)
