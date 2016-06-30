---
layout: post
title: "小玩意：如何让linux上挂死的进程重启？"
date: 2015-11-11 13:52:06
author: 伊布
categories: tech
tags: linux, keepalived
cover:  "/assets/instacode.png"
---

需求是这样的：我们在linux服务器上有一个采集进程，担心该进程出现故障挂死或者被人误杀，这种情况下需要能自动重启。使用peacemaker这样的分布式管理工具可以做到进程的监控，但毕竟体量较大，部署也稍嫌麻烦。
其实，使用keepalived就可以满足这种需求，部署起来也很简单，做个记录供以后查阅。

### 1、安装keepalived

### 2、配置keepalived检测
修改/etc/keepalived/keepalived.conf

```
vrrp_script check_dtm {
    script "/etc/keepalived/check_dtm.sh"
    interval 1
    weight -5
    fall 3
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
    }
    track_script {
       check_dtm
    }
}
```

如上配置，对VI_1实例配置track脚本，每秒检测一次，实际检测的脚本是check_dtm.sh。
由于我们只是使用keepalived的check功能，所以virtual_address和virtual_server的功能都不需要，相关配置全部删除。

### 3、配置检测脚本`check_dtm.sh`
```
#!/bin/bash
ps aux|grep dtmonitor|grep java

if [ $? != 0 ] ; then
    echo "dtmonitor is down, try to restart."
    bash /opt/dtmonitor/monitor/start.sh
fi
```

真正做到重启的地方。简单来说，检查进程是否还在（当然可以做的粒度更准确一些，例如定时写一些文件之类），如果进程没了，则调用采集进程的启动脚本，尝试重启。

### 4、采集进程的启动脚本。
在采集进程的目录中（即/opt/dtmonitor/monitor/）编辑start.sh文件：

```
#!/bin/bash

CURDIR="`dirname $0`"
java -jar $CURDIR/dtmonitor.jar &
echo "dtmonitor is started."
```

注意当前目录的切换。

如上，启动keepalived服务后，杀死dtmonitor进程，可以观察到1s左右dtmonitor进程被keepalived服务重启了。







