---
layout: post
title: "Spark（四）：python编程示例"
date: 2016-04-07 08:11:11
author: 伊布
categories: tech
tags: spark
cover:  "/assets/instacode.png"
---

下面以一个简单的例子，介绍下如何用python编程，并提交到yarn上执行。


### 环境准备

SparkContext是spark编程的基石，后面的SqlConext等等都是基于SparkContext。它作为python的lib，在pyspark库中提供，同时它还依赖py4j，所以我们要做的第一件事就是修改系统的python路径，把它俩加进去：

```
cd /usr/local/lib/python2.7/dist-packages/
echo "/home/dtdream/spark/spark-1.6.1-bin-hadoop2.6/python/"  >> spark.pth
echo "/home/dtdream/spark/spark-1.6.1-bin-hadoop2.6/python/lib/py4j-0.9-src.zip"  >> spark.pth
```

上面的路径替换为你放spark的实际路径。

> 不推荐直接在./bin/pyspark来做处理，它比较重，并且掩盖了sc的创建过程。其实作为一个应用，需要的只是pyspark库，官方管这种叫做“self-contained”。

### 编码

我的例子非常简单，将一个csv文件放到HDFS上，计算下有多少行，打印下第一行（官网的例子）。代码如下(1.py)：

```
from pyspark import SparkContext
if __name__ == "__main__":
    sc = SparkContext(appName="test1")
    textFile = sc.textFile("/test.csv")
    print textFile.count()
    print textFile.first()
    sc.stop()
```

### 集群上执行

生产环境上，任务都是在YARN上执行的，所以需要把这个任务submit上去：

```
{your_spark_home}/bin/spark-submit --master yarn 1.py
```

在yarn上可以看到这个application的记录。当然需要先配置好YARN需要的配置文件，具体可以参考第二篇文章中SPARK ON YARN部分。




