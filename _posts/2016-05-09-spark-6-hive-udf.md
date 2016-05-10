---
layout: post
title: "Spark（六）：一个Hive UDF编码问题的解决记录"
date: 2016-05-09 08:11:11
author: 伊布
categories: tech
tags:
- hive
- spark
cover:  "/assets/instacode.png"
---

> todo.....


spark的thrift server可以提供类似hive的体验，用户可以通过hive的JDBC连接到thrift server上。

1、UDF的不同类型及区别
2、需求：ip2region
3、UDF的执行
每次执行都是一次反射，包括init和evaluate
4、如何提升性能



