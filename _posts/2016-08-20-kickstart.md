---
layout: post
title: "如何为linux集群批量装机？"
date: 2016-08-20 00:11:12
author: 伊布
categories: tech
tags: kickstart
cover:  "/assets/instacode.png"
---

如何为linux集群批量装机？

U盘安装？古老的光盘安装？一两台还可以勉强接受，机器一多，时间会很长，还特别费工夫。对于服务器来说，一般考虑的方法是PXE安装，然后搭配kickstart自动装机。整个过程如下图：

![pxe+kickstart](http://www.centoscn.com/uploads/allimg/141215/012025B01-1.jpg)

实战中我们可以使用cobbler做集中管理，对cobbler的搭建感兴趣可以参考[Setup PXE Boot Environment Using Cobbler On CentOS 6.5](https://www.unixmen.com/setup-pxe-boot-environment-using-cobbler-centos-6-5/)，相对来说比较容易。

但这里有个问题。每台机器的网卡地址是DHCP分配的，那么我无法知道到底哪台服务器对应哪个IP地址，一旦有服务器发生了故障，没办法快速找到它。当然如果管理网络配置好了的话可以在PC上一个个远程看过去，但还是比较繁琐。

有什么办法能标识一台服务器呢？

接触过硬件服务器的同学会知道，服务器出厂之后都会有一个序列号，该序列号每个厂家的命名方式不同，有的纯数字，有的数字字母结合，但一定是唯一的，并且会贴在服务器上，那么只要能够根据序列号来分配IP地址，就能满足我们的核心需求了。

但是现实是残酷的。IP地址是DHCP服务器分配的，但是DHCP服务器最多只能根据MAC地址分配IP。序列号用不了了，用MAC地址怎么样？但是服务器出厂的时候并没有提供其MAC地址，可行的办法是先用显示器一台台连上服务器，记下来MAC地址，贴到服务器上，然后再……

算了，还是谈谈拯救地球的事吧。

---

回想下，只要能够在kickstart装机的时候拿到本机的序列号，然后从某个地方（比如rest服务）根据序列号取得IP地址、掩码、网关，并配置下去，是不是就解决问题了？

为了实现这个方案，我们需要解决如下几个问题。

- 提供一个rest服务，实现序列号到IP地址记录的http增删改查
- 提供一个命令行工具，把用户填写的服务器配置信息POST到rest服务中，生成装机任务；或者提供web服务，体验更佳
- kickstart模板中找个地方调用curl从rest服务器中取得IP配置，并配置到目标操作系统中

### rest服务(snservice)

由于需要的rest api一共没几个，所以我没有采用Spring之类的重型框架，而是采用了[Python Flask](http://docs.jinkan.org/docs/flask/)这种比较轻灵的框架。

我希望rest服务重启后，数据还能保存，所以需要一个持久化的数据库。这里我选用了mysql，使用flask_sqlalchemy来处理mysql数据的读取、插入、修改、删除。

代码不长，直接贴了。

```python
#!/usr/bin/python
from flask import Flask, jsonify
from flask import request
from flask import make_response
from flask_sqlalchemy import SQLAlchemy

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'mysql+pymysql://root:password@127.0.0.1/snservice'
app.config['SQLALCHEMY_TRACK_MODIFICATIONS']=True
db = SQLAlchemy(app)
db.create_all()


class Server(db.Model):
    sn = db.Column(db.String(80), primary_key=True)
    hostname = db.Column(db.String(64), unique=True)
    ip = db.Column(db.String(32), unique=False)
    netmask = db.Column(db.String(32), unique=False)
    gw = db.Column(db.String(32), unique=False)

    def __init__(self, sn, hostname, ip, netmask, gw):
        self.sn = sn
        self.hostname = hostname
        self.ip = ip
        self.netmask = netmask
        self.gw = gw

    @property
    def serialize(self):
       """Return object data in easily serializeable format"""
       return {
           'sn'       : self.sn,
           'hostname' : self.hostname,
           'ip'       : self.ip,
           'netmask'       : self.netmask,
           'gw'       : self.gw,
       }


@app.route('/')
def index():
    return "SN Service is running:)"


@app.route('/api/v1/servers/<string:server_sn>', methods=['GET'])
def get_task(server_sn):
    server = Server.query.filter_by(sn=server_sn).first()
    jserver = {
        'sn' : server.sn,
        'hostname' : server.hostname,
        'ip' : server.ip,
        'netmask' : server.netmask,
        'gw' : server.gw
    }
    return jsonify(jserver)


@app.route('/api/v1/servers', methods=['GET'])
def get_tasks():
    server = Server.query.all()
    return jsonify([i.serialize for i in server])


@app.route('/api/v1/servers', methods=['POST'])
def create_task():
    if not request.json or not 'sn' in request.json or not 'ip' in request.json or not 'netmask' in request.json or not 'gw' in request.json:
        return jsonify({'Error': 'Param error'})
    server = Server(request.json['sn'],request.json['hostname'], request.json['ip'], request.json['netmask'], request.json['gw'])
    db.session.merge(server)
    db.session.commit()
    jserver = {
        'sn' : server.sn,
        'hostname' : server.hostname,
        'ip' : server.ip,
        'netmask' : server.netmask,
        'gw' : server.gw
    }
    return jsonify(jserver), 201


@app.route('/api/v1/servers/<string:server_sn>', methods=['DELETE'])
def delete_task(server_sn):
    server = Server.query.filter_by(sn=server_sn).first()
    if not server:
        return jsonify({'Error': 'Not found'})
    db.session.delete(server)
    db.session.commit()
    jserver = {
        'sn' : server.sn,
        'hostname' : server.hostname,
        'ip' : server.ip,
        'netmask' : server.netmask,
        'gw' : server.gw
    }
    return jsonify(jserver), 201

@app.errorhandler(404)
def not_found(error):
    return make_response(jsonify({'error': 'Not found'}), 404)


if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080,debug=True)
```

mysql表的创建。

```
CREATE TABLE `server` (
  `sn` varchar(80) NOT NULL,
  `hostname` varchar(64) NOT NULL,
  `ip` varchar(32) NOT NULL,
  `netmask` varchar(32) NOT NULL,
  `gw` varchar(32) NOT NULL,
  PRIMARY KEY (`sn`)
);
```

提一下SQLAlchemy如何修改mysql的数据。由于我的表主键是sn（即同一台服务器只允许有一个IP配置），如果仍然使用add，会报主键冲突，写入失败。SQLAlchemy做的比较好，可以使用merge代替add，这样首次插入数据可以直接写入，而后续同主键数据更新也可以自动完成update。

### 命令行工具

要求用户在模板文件中填写各个服务器SN和IP信息，命令行工具会解析该文件，并将各个服务器的信息POST到snservice去。代码比较简单，就不贴了。后面会做一个web前端，更方便。

### kickstart配置IP

这里的坑比较多，一一道来。

我们在走读kickstart的模板文件时会看到其中会有网络相关的信息:

```
Network information
network --bootproto=dhcp --device=eth0 --onboot=on
```

意思比较明确，配置eth0的地址为DHCP方式获取，并且开机启动。由于我们的服务器是双万兆聚合，所以配置会改为这样：

```
network --device team0 --activate --bootproto static --ip=192.168.x.1 --netmask=255.255.255.0 --gateway=192.168.x.254 --nameserver=1.2.3.4  --teamslaves="eno1'{\"prio\": -10, \"sticky\": true}',eno2'{\"prio\": 100}'" --teamconfig="{\"runner\": {\"name\": \"lacp\"}}"
```

使用这样的配置为单台装机是没问题的，聚合可以正常生效。但我们的需求是为十几台或者几十台机器批量装机，ks模板不能像上面这些写死的。那么比较容易想到的办法是，在ks模板中调用dmidecode拿到sn后，再curl从snservice中拿到IP地址，以变量的方式丢给上面的network配置，在装机时变量替换就可以了。

但事实是，kickstart模板，只有在%pre和%post中可以使用变量（可以认为这时拥有了shell）；而network是跟这些货在一块的：

```
# System keyboard
keyboard us
# System language
lang en_US
```

这个过程不允许使用变量。就算在%pre中定义了的变量，在这个过程中也用不了。

好吧。我可以在%post中做。

```
%post
set -x -v
exec 1>/root/ks-post.log 2>&1

SN=`dmidecode -s system-serial-number`
_curl="http://{snservice ip}:8080/api/v1/servers/"$SN
json=`curl $_curl`
IP=`grep -o "\"ip\"\s*:\s*\"\([0-9]\{1,3\}\.\)\{3\}[0-9]\{1,3\}\+\"\?" <<<"$json" | sed -n -e 's/"//gp' | awk -F':' '{print $2}'`
GW=`grep -o "\"gw\"\s*:\s*\"\([0-9]\{1,3\}\.\)\{3\}[0-9]\{1,3\}\+\"\?" <<<"$json" | sed -n -e 's/"//gp' | awk -F':' '{print $2}'`

ETH1=`nmcli d s |awk '{print $1}' | sed -n '2p'`
ETH2=`nmcli d s |awk '{print $1}' | sed -n '3p'`
nmcli connection add type team con-name team0 ifname team0 config '{"runner":{"name":"lacp"}}'
nmcli connection add type team-slave con-name team0-port1 ifname $ETH1 master team0
nmcli connection add type team-slave con-name team0-port2 ifname $ETH2 master team0

nmcli connection modify team0 ipv4.addresses $IP/24 ipv4.gateway $GW
nmcli connection modify team0 ipv4.method manual
nmcli connection up team0
%end
```

很完美对吧。但是实际装机后发现，重启以后，centos7的/etc/sysconfig/network-scripts里压根就没有nmcli生成的配置文件，但/root/ks-post.log里又的确记录了nmcli成功执行的记录。

这里就需要说说kickstart装机的过程了。kickstart时运行的操作系统看到的实际是一个内存文件系统，而重启后进入的centos7的文件系统在kickstart装机时会挂载在/mnt/sysimage下（可以在kickstart的时候接上显示器进到shell去看）。所以上面的原因是nmcli生成的配置文件只写到了内存文件系统中，重启后在centos7上自然就看不到了。

但这里是有疑问的。%post不带参数，kickstart会chroot到centos7的目录下（即/mnt/sysimage），我在这里试过将ip信息echo到/root/info里，重启后是可以看到的，那么nmcli是否也应该只看到chroot以后的centos7的文件系统？从现象上来看nmcli还是突破了chroot的限制，那么chroot还怎么保证文件系统的安全性呢？后面有时间可以再看看chroot。

疑问先放一放，现在我们知道原因了，那么只要找个时机把生成的ifcfg-*文件拷贝到/mnt/sysimage/etc/sysconfig/network-scripts下就可以了。

%post有如下用法：

- %post --nochroot
- %post --interpreter /usr/bin/python
- %post --log /path/to/logfile
- %post

其中nochroot可以让我们看到完整的内存文件系统，所以只需要再在ks模板里加一段：

```
%post --nochroot
set -x -v
exec 1>/mnt/sysimage/root/ks-post-nochroot2.log 2>&1

cp -f /etc/sysconfig/network-scripts/ifcfg-* /mnt/sysimage/etc/sysconfig/network-scripts/
%end
```

看上去很完美了。但是实际装机后发现，两个网口的配置文件并不一样。我们是用nmcli来生成的，正常来说除了网口的名字、UUID以外，配置应该是一样的，出现不同应该是在%post之后还有一个步骤修改了网络的配置。

接上显示器，观察下%post之后kickstart打的日志信息，可以在post install日志后看到这么一段：

```
xx post install xx
..
Writing network configuration
..
Creating users
..
```

显然是它捣的鬼。查了下anaconda的代码，这个过程的代码在pyanaconda/install.py，会在装机到%post之后还会再做一些操作系统的配置，具体到网络，就是anaconda会读上面提到的ks模板里的Network information，并将配置写到第一个网口的配置信息里，所以最终我们看到两个网口的配置不一样。解决方法也比较简单，只要将Network information注释掉就可以了。

好了，是不是完美了呢？在其中一台双万兆口的服务器上测试的确没有问题了，但是当我换到另外一台双千兆口的服务器上测试，发现网络配置有问题，第一块物理网卡会通过DHCP申请到地址！而这块网卡是team0的一个成员口，正常来说不应该有地址。

原因是什么呢？由于对lacp协议不太了解，没有查的很仔细，直接说结论：由于上面把Network information注释掉了，但是anaconda最后还是会走一把网络配置（怎么这么拗呢你这孩子），其默认设置网卡的onboot熟悉是on的，因此最后生成的网卡1的配置带了ONBOOT=yes，可能这过程中造成了网卡1提前走了dhcp流程先申请到了地址。

解决办法是修改Network information，增加一条无差别的onboot=off。

```
# Network information
network --bootproto=dhcp --onboot=off
```

测试通过。

是不是真的完美了吗？其实并没有。细心的同学会发现snservice中其实我是有写hostname的，我希望装机以后hostname直接生效。但是即使在%post过程中将配置写到了/etc/hostname中，anaconda之后在处理Network information的时候，还是会用localhost覆盖掉hostname，没有找到合适的解决办法，如果您有方案，请告诉我。









