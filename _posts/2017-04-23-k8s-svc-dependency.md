---
layout: post
title: "kubernetes如何解决服务依赖呢？"
date: 2017-04-23 00:11:12
author: 伊布
categories: tech
tags: kubernets
cover:  "/assets/instacode.png"
---

* TOC
{:toc}

原文链接[在此](https://blog.giantswarm.io/wait-for-it-using-readiness-probes-for-service-dependencies-in-kubernetes/)。写的比较通俗易懂，做个笔记，有一些是我自己的理解。

在微服务的世界里，任何应用都需要注意，其所依赖的服务是会中断的。所以当应用发现某服务（如数据库）出现了故障，应该每隔一端时间去重试。而上层框架（如k8s）会检测到服务故障，并尝试恢复这个服务。

但在现实世界里，有些旧应用并没有处理这种情况，但我们还是希望能将他们也跑在微服务框架里，以期得到微服务的红利（例如应用故障重启），所以，需要定义服务依赖关系，从而保障旧应用启动时，它所依赖的服务已经ready。

解决方法是，微服务框架替应用等待其所依赖的服务（api, database, etc），当服务准备好时，框架才启动该应用。

### 如何知道Pod已经Ready

kubernetes提供了[Readiness Probe](http://kubernetes.io/docs/user-guide/pod-states/#when-should-i-use-liveness-or-readiness-probes)功能，用来探测Pod是否Ready。

Pod在Readiness Probe成功之前，不会接受任何流量；具体以Service来说，在Pod的Readiness Probe成功之前，Kubernetes不会将该Pod作为Serivce的Endpoint（注意，这是下面initContainer的基础）。Probe是kubernet API提供的功能，服务不需要做任何改变就可以支持。下面是一个readiness Probe的例子：

```
readinessProbe:
  httpGet:
    path: /login
    port: 3000
```


### 延迟应用的部署

我们已经解决了依赖服务是否Ready的问题，但还需要解决应用如何延迟部署；具体来说，如何让应用感知到其所依赖的服务已经Ready，但不必修改应用，当然还是需要框架来帮忙。

Kubernetes提供了init Container，它会在应用Pod启动之前启动，并且在init Container结束之前，应用Pod不会启动。所以，顾名思义，init Container用来初始化，在这里主要用来阻塞应用Pod的启动。（你将会看到在其他例子里initContainer还有一些其他的用法，例如可以用来修改与应用Pod共享的Volume，从而使相同的镜像具有不同的行为。initContainer很像打开了潘多拉的盒子）

```
annotations:
	pod.beta.kubernetes.io/init-containers: '[
		{
			"name": "wait-for-endpoints",
			"image": "giantswarm/tiny-tools",
			"imagePullPolicy": "IfNotPresent",
			"command": ["fish", "-c", "echo \"waiting for endpoints...\"; while true; set endpoints (curl -s --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt --header \"Authorization: Bearer \"(cat /var/run/secrets/kubernetes.io/serviceaccount/token) https://kubernetes.default.svc/api/v1/namespaces/monitoring/endpoints/grafana); echo $endpoints | jq \".\"; if test (echo $endpoints | jq -r \".subsets[]?.addresses // [] | length\") -gt 0; exit 0; end; echo \"waiting...\";sleep 1; end"],
			"args": ["monitoring", "grafana"]
		}
	]'
```

上面是initContainer的定义，简单来说就是每隔1s去问问kubernetes，我依赖的这个服务(namespaces/monitoring/endpoints/grafana)，是不是ready了，如果没有我就再while true一会，如果ready了我initContainer的使命就结束了，应用Pod可以启动了。

### 结论

Readiness+initContainer，算是Kubernetes为服务依赖给出的一个解决办法。但，其实这是一个workaroud，实际上完全可以作为kubernetes的一个feature（回想一下，initContainer的事情完全是k8s系统层面的，跟具体应用无关），那为什么kubernetes不实现呢？

我理解，这可能是k8s作为一个微服务的框架，不希望引入“服务依赖”，而是希望应用能够处理这种情况。毕竟，服务依赖带来一个问题：解决了应用启动时依赖的问题，那么应用运行过程中，所依赖的服务又故障了怎么办呢？

---
然而，现实情况没这么美。我给mysql集群前加的keepalived集群，希望不要老是重启，使用了上面的这个方法。

创建时的确是可以阻塞keepalived直到mysql集群ready；但在整集群重启的时候，initContainer由于之前已经complete了，所以集群重启时并不会再去拉起initContainer，所以也就不存在这个阻塞点了，因此服务依赖失败。

不过tiny-tools提供了一个fish shell很不错，可以试试。

---
