---
layout: post
title: "小玩意：东芝FlashAir无线SD存储卡，单反相机的好伙伴"
date: 2017-06-09 00:11:12
author: 伊布
categories: life
tags: FlashAir
cover:  "/assets/instacode.png"
---

今年春节过后，把单反相机从老家带了回来。相机是多年前买的，放在老家好几年了，我哥说别放家里了，都快长毛了。

前几天有次傍晚背着相机去江边玩，拍了张照片。

![暮色](http://7xir15.com1.z0.glb.clouddn.com/IMG_0050.JPG)

拍完了想发出去，但是这个相机本身没有WiFi功能，照片只能回家倒到电脑，然后再从电脑倒到手机里才行。那时早就意兴阑珊了。

其实在无线技术如此成熟的今天，万物早已互联，只要给SD卡装个WiFi AP就可以了，像我买的东芝FlashAir无线SD存储卡就是这种。

下面我要开始盗图了。

![FlashAir](http://7xir15.com1.z0.glb.clouddn.com/571dc7d1N328ad161.jpg)

图是JD扒下来的。

我买的是32GB的（不包含图上的三个姑娘），价格上带WiFi功能的SD卡大约是不带的2倍。可能容量更大更划算一点，毕竟WiFi功能本身也就100块钱，不过现在第三版最大也就是32GB。

使用起来也很方便，只要插到相机里去开机给FlashAir SD卡供电，默认SD卡会启动一个WiFi AP，手机连接上去以后，打开手机的FlashAir客户端，就可以看到相机上拍摄的照片了。

如果你不愿意再去安装一个客户端，也可以用手机浏览器访问```http://flashair```，可以直接浏览图片，通过手机浏览器去保存图片，体验虽不如客户端好，但胜在省空间。

下面是安卓手机的说明，iPhone/iPad也是类似。


![使用说明](http://7xir15.com1.z0.glb.clouddn.com/54f6e1b1N72419af2.jpg)

唯一不是很亚克西的是，下载的速度有点慢。

记得改下FlashAir WiFi的密码，默认密码太容易被猜对了，到时候相机里陈老师拍的照片被人拖走就不好了。我除了改密码，还把SSID改成了```Cisco Nexus 7000```，一看就不好惹。

手机客户端做的很一般，密码修改功能有点反人类，以至于我改了以后就忘记密码了。不过不要慌，FlashAir SD卡还有电脑的客户端，而且不光有Windows平台的，还有Mac平台的。

可以说非常良心了。

FlashAir还支持**互联网直通模式**，打开以后，手机可以同时浏览SD卡文件和访问互联网（需要设置平时家里用的WiFi SSID和密码）。文案上说是桥接功能，我猜SD卡自己起了个客户端连上了家里的WiFi，然后做了个透传，即：

![互联网直通](https://flashair-developers.com/images/tutorials/advanced_tutorial_03_1.png)


并不是手机同时连了2个WiFi，不然就违反一夫一妻制了。

这个功能很方便，但是实际上用处不是太大，因为在外面的时候通常没有WiFi用，手机连上FlashAir的WiFi以后，上网时不会用手机的4G网络，但是SD卡又没有可用的*互联网*，手机仍然上不了网。

如果FlashAir一直开着WiFi AP，对相机的电池来说比较有挑战，所幸它支持WiFi自动超时的功能，过几分钟会自动关闭WiFi。也可以彻底关闭WiFi功能，但再打开就需要用电脑客户端了，在外面不带电脑的时候要慎重慎重。

![FlashAir](http://7xir15.com1.z0.glb.clouddn.com/flashair_overview.png)

总的来说东芝FlashAir无线SD存储卡物有所值，解决了旧相机照片及时分享的问题，值得推荐。

---

如果你以为FlashAir只是上面介绍的这些功能，那就小看它了。

其实FlashAir是一个小型嵌入式设备，只要供电，就具备了CPU、内存、存储、WiFi，甚至还有GPIO，可玩性还可以。

![](https://flashair-developers.com/images/assets/flashair-overview-en.png)

[FlashAir™ Developers](https://flashair-developers.com/zh/)提供了一些开发资料，下面简单看看都有啥。

### command.cgi

cgi提供的主要就是手机上的FlashAir客户端的文件下载、显示缩略图、配置修改、控制GPIO等等。

### upload.cgi

FlashAir不光支持下载，还支持上传文件。可以使用 upload.cgi 通过无线连接向FlashAir卡上传文件，其实也就是个文件服务器了。具体可以看看[这里](https://flashair-developers.com/zh/documents/tutorials/advanced/2/)。


### lua脚本

[支持lua脚本！](https://flashair-developers.com/zh/documents/tutorials/lua/4/)

感觉可以为所欲为了。

1、[上传facebook](https://flashair-developers.com/zh/documents/tutorials/lua/5/)  （假装可以上Facebook）

2、[上传Dropbox](https://flashair-developers.com/zh/documents/tutorials/lua/6/) （假装可以上Dropbox）

3、 自定义http server

我觉得还是自定义http server想象空间更大一些。使用Lua，FlashAir可以动态生成HTML内容，

网站上也给了几个具体的例子。

1、[树莓派（Raspberry Pi）与FlashAir的合作](https://flashair-developers.com/zh/documents/tutorials/users/1/)

使用智能手机通过wifi读写树莓派的`/boot`文件夹。

2、[利用FlashAir控制火车模型-GPIO](https://flashair-developers.com/zh/documents/tutorials/users/2/)

还记得可以FlashAir可以控制GPIO吗？通常模型使用按键来控制I/O的高低电平，而这个例子使用FlashAir的WiFi和command.cgi来控制GPIO。




---
