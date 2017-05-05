---
layout: post
title: "杀死一只小鲸鱼"
date: 2017-05-05 00:11:12
author: 伊布
categories: tech
tags: docker
cover:  "/assets/instacode.png"
---

![docker banner](http://7xir15.com1.z0.glb.clouddn.com/docker-banner2.jpg)

之所以会有这个主题，是因为这几天我给mysql集群前置了一个keepalived，为了方便也做成了docker镜像，丢给k8s来部署。但实际测试时发现，当停止或者删除keepalived容器后，网卡上还残余之前keepalived下发的virtual ip。

直接在宿主机上安装keepalived不会有这个问题。从keepalived源码上来看，它会接管linux发给它的SIGTERM信号，之后清理现场，包括下发给接口的虚IP，所以问题原因就比较简单了，容器在退出时，并没用将SIGTERM信号传递给keepalived进程，造成keepalived强制退出，最终virtual ip残留。

要解决这个问题，还需要从容器退出开始谈起。先挖个坑，晚上填。

### docker容器退出会收到哪些信号？

不同的命令，收到的信号不同。

#### docker stop

默认docker stop会向容器发送SIGTERM信号，并等待10秒；如果10秒后容器没有退出，则会发送SIGKILL信号，强制杀死该容器。我的keepalived容器就是收到信号10s后仍然存活，最后被SIGKILL杀死的，docker ps -a查看容器的STATUS，可以看到```Exited (137) 2 hours ago```。137=128+9，即退出原因是收到了信号9（即SIGKILL）。

如果你的应用在退出时要做的事情比较多，可以docker stop的时候宽限几天。

```
docker stop ----time=30 xxx
```

#### docker kill

默认docker kill会向容器发送SIGKILL信号，即直接杀死容器，不叨叨。不过你可以通过--signal=SIGXXX参数来修改docker kill发送的信号。注意docker kill和linux kill命令的默认行为不同，后者默认发送的是SIGTERM(类似docker stop)。docker kill和linux kill -9的行为是类似的。

#### docker rm -f

docker rm -f会发送SIGKILL杀死容器，并清理容器痕迹，类似docker kill && docker rm。

### 应用应该处理哪个信号？

新写的应用，建议捕捉SIGTERM，因为这是docker stop时默认发出的信号；旧有的应用，则需要一房一价，具体问题具体分析，例如NGINX需要SIGQUIT，Apache需要SIGWINCH。keepalived比较友好：

```
/* Terminate handler */
void
sigend(void *v, int sig)
{
	int status;

	/* register the terminate thread */
	thread_add_terminate_event(master);

	if (vrrp_child > 0) {
		kill(vrrp_child, SIGTERM);
		waitpid(vrrp_child, &status, WNOHANG);
	}
	if (checkers_child > 0) {
		kill(checkers_child, SIGTERM);
		waitpid(checkers_child, &status, WNOHANG);
	}
}

/* Initialize signal handler */
void
signal_init(void)
{
	signal_handler_init();
	signal_set(SIGHUP, sighup, NULL);
	signal_set(SIGINT, sigend, NULL);
	signal_set(SIGTERM, sigend, NULL);
	signal_ignore(SIGPIPE);
}
```

不管是SIGINT还是SIGTERM，都能sigend走清理。

### 应用是怎么接收到信号的

docker会将信号传给pid=1的进程。

换句话说，如果你的应用跑起来时pid不为1，那么就收不到docker发出来的信号，也就做不到gracefully退出了。因此，我们的目标就是要让需要gracefully退出的应用跑在pid=1的位置上。

举个例子。

先写个loop.sh。

```
#!/usr/bin/env bash
trap 'exit 0' SIGTERM
while true; do :; done
```

然后是Dockerfile。

```
FROM ubuntu:trusty
COPY loop.sh /
CMD /loop.sh
```

你可以试着build/run/stop一下，会发现docker stop需要10s才能停掉这个容器，说明loop.sh没有收到信号。为什么呢？我们知道信号是发给pid=1的进程，那么来看看容器里的情况：

```
docker exec 838 ps aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.0   4460   696 ?        Ss   05:26   0:00 /bin/sh -c /loop.sh
root         6 99.6  0.0  17968  2716 ?        R    05:26   0:29 bash /loop.sh
root         7  0.0  0.0  15580  2156 ?        Rs   05:26   0:00 ps aux
```

显然loop.sh收不到信号。

Dockerfile对于```CMD command param1 param2```这种形式的命令，具体执行的是```bash -c command param1 param2```，也就是上面我们看到的结果。

正确的写法是，使用```CMD ["executable","param1","param2"]```这种形式，例如上面：

```
FROM ubuntu:trusty
COPY [/loop.sh]
```

这样容器启动时，会直接执行loop.sh，不会在外面再套一层bash：

```
docker exec 638 ps aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1 91.8  0.0  17968  2864 ?        Rs   05:36   0:09 bash /loop.sh
root         6  0.0  0.0  15580  2032 ?        Rs   05:36   0:00 ps aux
```

这样应用就可以拿到docker发送的SIGTERM信号了。

再举个例子。

比如我的keepalived容器。由于容器启动时我需要做一些配置修改（keepalived.conf），需要把这些操作放在一个脚本里，所以我的CMD是：```CMD ["/opt/entry-point.sh"]```，在entry-point.sh中启动keepalived进程。

```
#!/bin/bash -x
#entry-point.sh
#replace variables in /etc/keepalived/keepalived.conf
/sbin/ipvsadm-restore < /etc/sysconfig/ipvsadm
/usr/sbin/keepalived --dont-fork --log-console --pid /keepalived.pid -D
tail -f /var/log/yum.log
```

entry-point.sh脚本的确是pid=1了，但keepalived进程的pid并不是1，因为它和上面的```CMD command```一样，是bash -c方式以entry-point.sh的子进程的身份执行的。正确的做法是什么呢？

要exec，不要fork：

```
#!/bin/bash -x
#entry-point.sh
#replace variables in /etc/keepalived/keepalived.conf
/sbin/ipvsadm-restore < /etc/sysconfig/ipvsadm
exec /usr/sbin/keepalived --dont-fork --log-console --pid /keepalived.pid -D
```

此时再来看容器内的进程，keepalived就是pid=1了，这时去docker stop容器，keepalived会捕捉到SIGTERM信号，然后清理掉virtual ip，gracefully的退出。

```
docker exec 576 ps aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.0 111660  4000 ?        Ss   09:41   0:00 /usr/sbin/keepalived --dont-fork --log-console --pid /keepalived.pid -D
root        18  0.0  0.0 113904  2952 ?        S    09:41   0:02 /usr/sbin/keepalived --dont-fork --log-console --pid /keepalived.pid -D
root        19  0.0  0.0 113780  2192 ?        S    09:41   0:04 /usr/sbin/keepalived --dont-fork --log-console --pid /keepalived.pid -D
root        24  0.0  0.0  35884  1456 ?        Rs   13:48   0:00 ps aux
```




---
Ref:

[Gracefully Stopping Docker Containers](https://www.ctl.io/developers/blog/post/gracefully-stopping-docker-containers/)

[LVS + Keepalived](https://www.server-world.info/en/note?os=CentOS_7&p=lvs&f=2)

---
