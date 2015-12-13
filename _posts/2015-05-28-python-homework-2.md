---
layout: post
title: "python学习作业（二）"
date: 2015-05-28 23:11:15
tags: python
author: 伊布
categories: tech
cover:  "/assets/instacode.png"
---


需求：生成纯文本格式的表数据，导入ODPS/ADS等。用户可以定义表的格式。
最早是想找一个类似DataFactory的工具来做，但今天问了下史进是自己用Java写的，走读了一遍逻辑不太复杂，于是花了些时间写了一个python版本的。

实现思路非常简单：配置文件使用json格式，python读取配置文件，按每一行的格式随机生成内容，每行生成后写入文件，最后关闭文件，结束。

几个有意思的点：

- json配置文件怎么写注释？json本身并没有定义注释，但是可以像本例一样加一个注释字段，占坑但不拉翔。
- 随机字符串的生成。`GenStr`的实现是从网上找到一个方法稍作修改：先随机生成若干字符，然后join。不知道是否有更高效的实现方法？
- 随机时间的生成，我这里的做法是取当前时间的浮点表示值，然后取比这个小的一个随机值，最后将其转为时间格式。
- 类似`switch case`的编码风格。python不支持switch，原因[见这里](https://docs.python.org/2/faq/design.html#why-isn-t-there-a-switch-or-case-statement-in-python)。我这里也是参考网上的一个做法，定义函数数组，效率比`if...elif...elif`应该高一点。


配置文件cfg.json：

```json
{
  "filename" : "test.txt",
  "table":
  {
    "lines" : "30",
    "_commet_for_columes" : "colume type support BOOLEAN, BIGINT, DOUBLE, STRING, DATATIME only.",
    "columes":"BIGINT, STRING, DATETIME, BOOLEAN, DOUBLE"
  }
}
```

生成数据gendata.py：

```python
import sys
import string
import json
import random
import time

staticChars = string.ascii_letters+string.digits
staticCharLen = len(staticChars)

def GenBoolean():
    return str(random.randint(0, 1))
def GenBigInt():
    return str(random.randint(0, sys.maxint))
def GenDouble():
    return str(random.uniform(0, sys.maxint))
def GenStr():
    length = random.randint(1, staticCharLen)
    return ''.join([random.choice(staticChars) for i in range(length)])
def GenDate():
    return time.strftime("%Y-%M-%d", time.localtime(random.uniform(0,time.time())))

funcs = {
    "BOOLEAN": GenBoolean,
    "BIGINT" : GenBigInt,
    "DOUBLE" : GenDouble,
    "STRING" : GenStr,
    "DATETIME" : GenDate
}

def GenData(ctype):
    return funcs[ctype.upper().strip()]()

def main():
    f = file("cfg.json")
    jscfg = json.load(f)

    lines = int(jscfg[u'table'][u'lines'])
    columes = str(jscfg[u'table'][u'columes']).split(',')

    filep = file(jscfg[u'filename'], 'w')
    while lines > 0:
        lines -= 1
        for type in columes:
            filep.write(GenData(type) + ',')
        filep.write('\n')
    filep.close()

if __name__ == "__main__":
    main()
```


