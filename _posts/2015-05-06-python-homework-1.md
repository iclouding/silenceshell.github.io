---
layout: post
title: "python学习作业（一）"
date: 2015-05-06 15:45:47
tags: python
author: 伊布
categories: tech
cover:  "/assets/instacode.png"
---


这个系列记录Python小组学习过程中完成的作业，python用的很不熟，希望做作业的过程中有进步。

本次作业题目：

|代码  |                     城市|
|----|----|
|CHXX0008  |                北京|
|CHXX0044    |              杭州|
|CHXX0116     |             上海|
|CHXX0502      |            海口|
|CHXX0037       |           广州|

任意输入上表内的城市名字，与今天的时间差(明天、后天、大后天)），输出中文天气情况。
天气数据从yahoo网站获取，不同城市输入不同代码，如杭州从这个网址获取：
http://xml.weather.yahoo.com/forecastrss?u=c&p=CHXX0044

用到的几个点：

- 使用requests库读取网页
- 使用beatifulSoup解析（正则表达式）
- 使用有道翻译的API做英译中。API需要注册一下:[注册地址](http://fanyi.youdao.com/openapi?path=data-mode)
申请成功后会提供API key和keyfroom，也会提示url的使用方法。我这里的doctype使用了json。
API key：12345678
keyfrom：dtdream
- 使用json解析有道翻译的返回结果

中文编码出了点问题，可以通过`#coding=utf-8`搞定。
代码如下：

```python
#!/usr/bin/env python
#coding=utf-8

import re
import requests
import bs4
import json
import urllib

citycodeArr = {'北京' : 'CHXX0008', '杭州' : 'CHXX0044', '上海' : 'CHXX0116', '海口' : 'CHXX0502', '广州' : 'CHXX0037', '唐>山' : 'CHXX0131'}

def trans(string):
  url = 'http://fanyi.youdao.com/openapi.do?keyfrom=dtdream&key=1972884851&type=data&doctype=json&version=1.1&q='+string
  try:
    page = urllib.urlopen(url)
    jsonStr = page.read()
    trans_res = json.loads(jsonStr)
    new = trans_res[u"translation"]
    print(new[0])
  except Error:
    print("出错了")

def checkWeather():
  city = raw_input("城市? ")
  after =input("几天后? ")

  if (city not in citycodeArr) or (after > 4):
    print("超出预报范围")
    return -1
  citycode = citycodeArr[city]
  reqUrl = "http://xml.weather.yahoo.com/forecastrss?u=c&p=" + citycode

  response = requests.get(reqUrl)
  soup = bs4.BeautifulSoup(response.text)
  weatherAll = soup.find_all(re.compile("yweather:forecast"))
  trans(weatherAll[after].get('text'))
  return 0

if __name__ == '__main__':
  checkWeather()
```

