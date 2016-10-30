---
layout: post
title: "国内真实好用的maven镜像"
date: 2016-10-30 00:11:12
author: 伊布
categories: tech
tags: zeppelin
cover:  "/assets/instacode.png"
---

不管是Google还是百度还是必应，去搜“国内maven镜像”，很多老PO文都会告诉你配置成oschina，然而oschina已经关闭了maven镜像服务了，sigh。之前一直忍着用公司搭建的maven镜像，速度倒是很不错，但是这个镜像最大的问题是，MD经常会有一些jar包取不下来，每次都是跟管理员反馈以后等很久才解决，不胜其烦。

这周找到简书上有篇[PO文](http://www.jianshu.com/p/4d5bb95b56c5)里提到现在阿里云也提供了maven镜像，立马换过来。注意原PO的mirrorOf是central，这样maven只有在包属于central库的时候才会用阿里云镜像，但其实阿里云的镜像还是[很全面的](http://maven.aliyun.com/nexus/content/repositories/?spm=0.0.0.0.Gagfio/)，所以我这里直接改为了'*'。

```xml
    <mirror>
      <id>alimaven</id>
      <name>aliyun maven</name>
      <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
      <mirrorOf>*</mirrorOf>
    </mirror>
```

换上以后，有种便秘终于拉出来了的感觉，你也不妨试试。
