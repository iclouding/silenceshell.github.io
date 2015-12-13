---
layout: post
title: "使用nginx+keepalived实现RESTful API服务器的负载均衡和高可靠性"
date: 2015-07-02 22:08:53
tags: nginx keepalived
author: 伊布
categories: tech
cover:  "/assets/instacode.png"
---

核心需求是我们有一个RESTful API的服务集群，需要能够保证不管是web服务故障还是服务器整体故障，外部访问不会间断，并且在整体运行正常的时候，需要能够负载均衡。
业界比较常见的几个负载均衡方案有haproxy, nginx, lvs。有关这仨的比较，可以看[这篇文章](http://www.csdn.net/article/2014-07-24/2820837)。我这里选择的方案是nginx+keepalived。nginx做反向代理，可以实现负载均衡，如果后端的web服务故障了，nginx可以实现切换；但nginx本身存在单点故障，需要通过keepalived监测实现nginx的切换。


整体结构图
![](http://7xir15.com1.z0.glb.clouddn.com/nginx反向代理.png)

##1、设置nginx.repo：
我用的操作系统是centos6.5，如下：

```bash
[nginx]
name=nginx repo
baseurl=http://nginx.org/packages/centos/$releasever/$basearch/
gpgcheck=0
enabled=1
```

如果是RHEL:

```bash
[nginx]
name=nginx repo
baseurl=http://nginx.org/packages/rhel/$releasever/$basearch/
gpgcheck=0
enabled=1
```

##2、安装nginx、keepalived

```bash
$ yum install nginx
$ yum install keepalived
```

安装完毕后两个服务都是停止的，需要start并加到系统启动服务中。

```bash
$ chkconfig  nginx on
$ chkconfig  keepalived on
```

nginx启动后，默认会有一个http server，例如我这里访问的地址是`http://192.168.80.165`和`http://192.168.80.166`，两台服务器的地址。但实际上我不需要这俩web服务器，而是需要让nginx做反向代理，将http请求导引到我的RESTful API服务器上，配置下面会有提到。

##3、修改keepalived的配置文件
配置文件的路径是`/etc/keepalived/keepalived.conf`。
master：

```bash
vrrp_instance VI_1 {
    state MASTER
    interface eth2   #具体的网卡
    virtual_router_id 51
    priority 101
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.80.111
    }
}
```

slave：

```bash
vrrp_instance VI_1 {
    state MASTER
    interface eth4   #具体的网卡
    virtual_router_id 51
    priority 100     #比master小
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.80.111
    }
}
```

其他的virtual server等信息都删掉，那是给lvs用的。
配置完毕后，重启两台服务器上的keepalived进程，在master上可以看到我们配置的虚IP（ifconfig看不到）。将master上keepalived服务stop掉，可以看到虚IP跑到slave上了；再启动master上keepalived进程，虚IP会被master抢占回来，因为master的priority更大。

```bash
# ip add
...
2: eth2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 00:0c:29:73:f6:15 brd ff:ff:ff:ff:ff:ff
    inet 192.168.80.165/24 brd 192.168.80.255 scope global eth2
    inet 192.168.80.111/32 scope global eth2
```

##4、使用脚本检测nginx服务
上面的配置可以保证keepalived关闭（例如服务器故障）时，虚IP自动切换；但有的时候可能只是web服务故障了，我们希望的是keepalived检测服务的状态，并且能够自动切换。这种情况可以用脚本来检测nginx服务状态，根据检测结果调高或调低vrrp的优先级，达到虚IP切换的目的。
新建一个探测脚本：check_nginx.sh

```bash
#!/bin/bash
netstat -antp|grep nginx
exit $?
```

修改keepalived.conf：

```bash
vrrp_script check_succ {
    script "/etc/keepalived/check_nginx.sh"
    interval 1
    weight -5
    fall 3
}
vrrp_script check_fail {
    script "/etc/keepalived/check_nginx.sh"
    interval 1
    weight 5
    rise 2
}
vrrp_instance VI_1 {
...
    track_script {
       check_succ
       check_fail
    }
```

据说探测成功或失败了以后只会改一次优先级，所以不要担心不停探测优先级一直增长的问题。
简单说明下，上面的脚本简单的检查了nginx是不是还在监听端口，如果发现不是（例如主的nginx被stop），则priority-5，vrrp通告出去后，备发现自己的优先级更高，vrrp切换，备抢占虚IP，此时访问的nginx就是备上的了；等到主nginx重新启动后，脚本检查端口已在监听，则priority+5，vrrp切换，主会重新抢占虚IP，达到HA的目的。

##5、配置nginx
上面配置完keepalived后，HA的功能完成了，但是用户只能访问一个服务器，对于有多个web容器的情况就无能为力了，这时候需要nginx出马。
nginx在我们的组网里实际是一个loadbalance的角色，将用户的请求分发给不同的server（即upstream）。由于我们后端服务器监听的是8443 ssl端口，所以步骤稍微复杂一点。
###5.1 配置nginx
CENTOS6.5的nginx配置是在`/etc/nginx/conf.d/default`，我直接给出配置（基本是照抄了以升的说明）：

```bash
upstream dxt
{
        server 192.168.80.165:8443;   #负载分担的两个服务器,
        server 192.168.80.166:8443;   #也就是我这里的rest api服务，分在两台服务器上
}

server {
    listen       443 ssl;			#由于nginx和restapi服务在同一台服务器上，需要使用不同的端口
    server_name  192.168.80.111;	#虚IP

    root html;
    index index.html index.htm;

    ssl on;							#配置ssl
    ssl_certificate server.crt;
    ssl_certificate_key server.key;
    ssl_session_timeout 5m;

    ssl_protocols SSLv3 TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers "HIGH:!aNULL:!MD5 or HIGH:!aNULL:!MD5:!3DES";
    ssl_prefer_server_ciphers on;

    location /api/ {				#只处理/api这个路径
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_set_header X-NginX-Proxy true;

        proxy_pass https://dxt;		#指定upstream
        proxy_redirect off;
    }
}
```

###5.2 配置ssl需要的server.crt、server.key
使用ssl还需要两个证书文件，这里也按照以升给出的方法生成。

```bash
# cd /etc/nginx
# openssl genrsa -des3 -out server.key 1024
# openssl req -new -key server.key -out server.csr
# cp server.key server.key.org
# openssl rsa -in server.key.org -out server.key
# openssl x509 -req -days 365 -in server.csr -signkey server.key -out server.crt
```

配置后重启nginx，浏览器访问`https://192.168.80.111:443/api/`，应该可以看到restapi的信息了。

###5.3 坑
！！但是这里有个坑，如果你跟我一样也是用的centos6.5，会发现浏览器返回的是这样的：
![nginx错误信息](http://7xir15.com1.z0.glb.clouddn.com/nginx错误.png)
说明还停留在nginx上，反向代理失败了。
去查nginx的日志`/var/log/nginx/error.log`，看到下面的信息：

```bash
2015/07/02 09:52:12 [error] 15053#0: *8 SSL_do_handshake() failed (SSL: error:100AE081:elliptic curve routines:EC_GROUP_new_by_curve_name:unknown group error:1408D010:SSL routines:SSL3_GET_KEY_EXCHANGE:EC lib) while SSL handshaking to upstream, client: 192.168.80.1, server: 192.168.80.111, request: "GET /api/ HTTP/1.1", upstream: "https://192.168.80.166:8443/api/", host: "192.168.80.111"
```

查了一下，说这是centos6.5上默认openssl版本的错误，需要更新openssl的版本。可以查看[这篇文章](http://zh.hortonworks.com/community/forums/topic/ambari-agent-registration-failure-on-rhel-6-5-due-to-openssl-2/)，或者你懒的看，直接`yum update openssl`即可。
升级以后的版本应该是这个：

```bash
# rpm -aq|grep openssl
openssl-1.0.1e-30.el6.11.x86_64
```

升级完毕后再重启一下nginx，现在访问虚IP，就能看到restapi的信息了。如果你用POSTMAN这种restapi客户端打几次请求，从rest server日志里可以看到是轮询访问不同的rest server。
![](http://7xir15.com1.z0.glb.clouddn.com/虚IP.png)

----

通过上面的keepalived和nginx的配置，我们完成了开始预设的要求：
1. rest server能够负载分担；
2. 某rest server进程故障，可以由nginx剔除；
3. nginx故障，keepalived可以切换虚IP到正常nginx，由新的nginx继续负载分担；nginx故障恢复，切换回原来的主server；
4. 整设备故障，vrrp超时切换虚IP到正常服务器；故障恢复，回切。
