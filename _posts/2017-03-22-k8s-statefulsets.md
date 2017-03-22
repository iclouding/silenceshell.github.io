---
layout: post
title: "kubernets: StatefulSets Basics"
date: 2017-03-22 00:11:12
author: 伊布
categories: tech
tags: kubernets
cover:  "/assets/instacode.png"
---

[原文在这](https://kubernetes.io/docs/tutorials/stateful-application/basic-stateful-set/)。部分涉及到PV的地方做了修改。


### Objectives

StatefulSets是为了有状态的应用和分布式系统设计的。然而有状态应用、分布式系统的管理是一个宽泛、复杂的主题。为了演示StatefulSet的基础特性，以及避免混淆前后主题，你会使用StatefulSets部署一个简单的web应用。

完成这个教程后，你会熟悉：

- 如何创建一个StatefulSets
- StatefulSets如何管理它的Pods
- 如何删除一个StatefulSets
- 如何扩展一个StatefulSets
- 如何更新一个StatefulSets中的Pod的容器镜像

### Before you begin

开始教程之前，你需要熟悉如下k8s概念：

- Pods
- Cluster DNS
- Headless Services
- PersistentVolumes
- PersistentVolume Provisioning
- StatefulSets
- kubectl CLI


本教程假设你的集群能够动态提供持久化存储。否则，你需要在开始教程之前手动提供5个1GB的卷。

【注】Headless Service的说明见[这里](http://www.datastart.cn/tech/2017/03/22/k8s-headless-service.html)

```
# kubectl describe  service nginx
Name:                   nginx
Namespace:              default
Labels:                 app=nginx
Selector:               app=nginx
Type:                   ClusterIP
IP:                     None
Port:                   web     80/TCP
Endpoints:              10.244.2.17:80,10.244.2.18:80
Session Affinity:       None
No events.
```


### Creating a StatefulSet

用下面的例子来创建一个StatefulSet。它创建了一个名为nginx的Headless Service，用以控制StatefulSet web的域。

```yml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
---
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.11
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
          volumes:
            - name: www
              hostPath:
                path: /mydir
```

【注】：可能你跟我一样，没有一个PersistVolume，所以我贴心的把原文里的持久化存储，改为了普通的volume。当然你需要在宿主机的/mydir里放一个index.html，里面随便写点什么都可以。另外要求的image我也从grc.io的nginx-slim改为了docker hub的nginx:1.11。

下载并保存。你需要2个terminal。在第一个terminal里，用kubectl来观察StatefulSets的容器的创建过程：

```
kubectl get pods -w -l app=nginx
```

在第二个terminal里，用kubectl create来创建web.yml里定义的Headless Service和StatefulSets。

```
kubectl create -f web.yaml
service "nginx" created
statefulset "web" created
```

上面的命令会创建2个Pods，每个Pods中都运行着一个NGINX webserver。查看下nginx Service和 web SatatefulSets是否已经创建成功了。

```
kubectl get service nginx
NAME      CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
nginx     None         <none>        80/TCP    12s

kubectl get statefulset web
NAME      DESIRED   CURRENT   AGE
web       2         1         20s
```

**Ordered Pod Creation**

对于一个N副本的StatefulSet，其Pods在部署时是以{0..N-1}的顺序依次创建的。这一点可以从第一个窗口的输出信息里来验证。

```
kubectl get pods -w -l app=nginx
NAME      READY     STATUS    RESTARTS   AGE
NAME      READY     STATUS    RESTARTS   AGE
web-0     0/1       Pending   0          0s
web-0     0/1       Pending   0         0s
web-0     0/1       ContainerCreating   0         0s
web-0     1/1       Running   0         19s
web-1     0/1       Pending   0         0s
web-1     0/1       Pending   0         0s
web-1     0/1       ContainerCreating   0         0s
web-1     1/1       Running   0         18s
```

注意，直到web-0状态启动后达到Running状态之后，web-1才开始启动。

### Pods in a StatefulSet

和其他控制器中的Pods不一样，StatefulSets中的Pods具有唯一的原生索引，以及稳定的网络标识。

**Examining the Pod’s Ordinal Index**

Get the StatefulSet’s Pods.

```
kubectl get pods -l app=nginx
NAME      READY     STATUS    RESTARTS   AGE
web-0     1/1       Running   0          1m
web-1     1/1       Running   0          1m
```

As mentioned in the StatefulSets concept, the Pods in a StatefulSet have a sticky, unique identity. This identity is based on a unique ordinal index that is assigned to each Pod by the Stateful Set controller. The Pods’ names take the form <statefulset name>-<ordinal index>. Since the web StatefulSet has two replicas, it creates two Pods, web-0 and web-1.

Pods有唯一标识。

**Using Stable Network Identities**

Each Pod has a stable hostname based on its ordinal index. Use kubectl exec to execute the hostname command in each Pod.

每个Pod的hostname是稳定不变的，hostname是根据原始索引生成的。

```
for i in 0 1; do kubectl exec web-$i -- sh -c 'hostname'; done
web-0
web-1
```

用kubectl run启动一个容器，通过容器提供的nslookup命令来查询Pods的名字，可以查询各Pods在集群内的DNS地址。

```
kubectl run -i --tty --image busybox dns-test --restart=Never --rm /bin/sh
# nslookup web-0.nginx
Server:    10.0.0.10
Address 1: 10.0.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-0.nginx
Address 1: 10.244.1.6

# nslookup web-1.nginx
Server:    10.0.0.10
Address 1: 10.0.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-1.nginx
Address 1: 10.244.2.6
```

The CNAME of the headless service points to SRV records (one for each Pod that is Running and Ready). The SRV records point to A record entries that contain the Pods’ IP addresses.

headless service的CNAME（别名）指向SRV records（每个Running&Ready状态的Pod一条）。SRV records指向A record（A record包含Pod的IP地址）。

In one terminal, watch the StatefulSet’s Pods.

```
kubectl get pod -w -l app=nginx
```

In a second terminal, use kubectl delete to delete all the Pods in the StatefulSet.

```
kubectl delete pod -l app=nginx
pod "web-0" deleted
pod "web-1" deleted
```

Wait for the StatefulSet to restart them, and for both Pods to transition to Running and Ready.

```
kubectl get pod -w -l app=nginx
NAME      READY     STATUS              RESTARTS   AGE
web-0     0/1       ContainerCreating   0          0s
NAME      READY     STATUS    RESTARTS   AGE
web-0     1/1       Running   0          2s
web-1     0/1       Pending   0         0s
web-1     0/1       Pending   0         0s
web-1     0/1       ContainerCreating   0         0s
web-1     1/1       Running   0         34s
```

Use kubectl exec and kubectl run to view the Pods hostnames and in-cluster DNS entries.

```
for i in 0 1; do kubectl exec web-$i -- sh -c 'hostname'; done
web-0
web-1

kubectl run -i --tty --image busybox dns-test --restart=Never --rm /bin/sh
nslookup web-0.nginx
Server:    10.0.0.10
Address 1: 10.0.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-0.nginx
Address 1: 10.244.1.7

nslookup web-1.nginx
Server:    10.0.0.10
Address 1: 10.0.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-1.nginx
Address 1: 10.244.2.8
```

The Pods’ ordinals, hostnames, SRV records, and A record names have not changed, but the IP addresses associated with the Pods may have changed. In the cluster used for this tutorial, they have. This is why it is important not to configure other applications to connect to Pods in a StatefulSet by IP address.

删除该StatefulSet的pod，StatefulSet Controller会自动重新拉起新的Pod。新的Pod的域名、SRV records、A record都没有变化，但是IP地址变了。所以其他应用不要直接使用StatefulSet的IP地址。

If you need to find and connect to the active members of a StatefulSet, you should query the CNAME of the Headless Service (nginx.default.svc.cluster.local). The SRV records associated with the CNAME will contain only the Pods in the StatefulSet that are Running and Ready.

如果你需要连接StatefulSet的活跃成员，你需要查询Headless Service的CNAME（nginx.default.svc.cluster.local）。跟这个CNAME关联的SRV records会只包含StatefulSet中处于Running&Ready状态的Pods。

If your application already implements connection logic that tests for liveness and readiness, you can use the SRV records of the Pods ( web-0.nginx.default.svc.cluster.local, web-1.nginx.default.svc.cluster.local), as they are stable, and your application will be able to discover the Pods’ addresses when they transition to Running and Ready.

如果你的应用已经实现了测试用的连接逻辑，你可以使用Pods的SRV records(web-0.nginx.default.svc.cluster.local, web-1.nginx.default.svc.cluster.local)，因为SRV record是稳定的；当Pods状态为Running&Ready状态时，你的应用可以发现Pods的IP地址。

**Writing to Stable Storage**

【注】原文使用了PersistVolume，由于我是使用的hostPath，所以，PV的部分我会跳过去。

The NGINX webservers, by default, will serve an index file at /usr/share/nginx/html/index.html. The volumeMounts field in the StatefulSets spec ensures that the /usr/share/nginx/html directory is backed by a PersistentVolume.

默认Nginx服务器会使用/usr/share/nginx/html/index.html作为其索引文件。StatefulSets中的volumeMounts段会保证/usr/share/nginx/html能够通过~~PersistentVolume~~hostPath得到备份。
【注】如果只是用hostPath，因为delete pod重新调度新pod后，可能会发生pod漂移，而新node上没有对应目录下的文件并不相同，因此会有问题。更好的做法是使用nodeSelector将该pod固定在某一节点。当然，如果你有PV，当我没说。

Write the Pods’ hostnames to their index.html files and verify that the NGINX webservers serve the hostnames.

```
for i in 0 1; do kubectl exec web-$i -- sh -c 'echo $(hostname) > /usr/share/nginx/html/index.html'; done
for i in 0 1; do kubectl exec -it web-$i -- cat /usr/share/nginx/html/index.html; done
web-0
web-1
```

【注】由于前面我没有用gcr.io的nginx-slim，所以你在执行第二条的curl命令时会遇到找不到curl命令的问题。没关系，你直接查看pod对应目录的文件就好了，跟curl是一样的，只不过curl可以顺带验证nginx服务是ok的。

Note, if you instead see 403 Forbidden responses for the above curl command, you will need to fix the permissions of the directory mounted by the volumeMounts (due to a bug when using hostPath volumes) with:

如果遇到403 Forbidden，说明目录权限有问题的，设置下就好了。

```
for i in 0 1; do kubectl exec web-$i -- chmod 755 /usr/share/nginx/html; done
before retrying the curl command above.
```

In one terminal, watch the StatefulSet’s Pods.

```
kubectl get pod -w -l app=nginx
```

In a second terminal, delete all of the StatefulSet’s Pods.

```
kubectl delete pod -l app=nginx
pod "web-0" deleted
pod "web-1" deleted
```

Examine the output of the kubectl get command in the first terminal, and wait for all of the Pods to transition to Running and Ready.

```
kubectl get pod -w -l app=nginx
NAME      READY     STATUS              RESTARTS   AGE
web-0     0/1       ContainerCreating   0          0s
NAME      READY     STATUS    RESTARTS   AGE
web-0     1/1       Running   0          2s
web-1     0/1       Pending   0         0s
web-1     0/1       Pending   0         0s
web-1     0/1       ContainerCreating   0         0s
web-1     1/1       Running   0         34s
```

再来一把删除pod重调度。

Verify the web servers continue to serve their hostnames.

```
for i in 0 1; do kubectl exec -it web-$i -- cat /usr/share/nginx/html/index.html; done
web-0
web-1
```

Event though web-0 and web-1 were rescheduled, they continue to serve their hostnames because the PersistentVolumes associated with their Persistent Volume Claims are remounted to their volumeMounts. No matter what node web-0 and web-1 are scheduled on, their PersistentVolumes will be mounted to the appropriate mount points.

虽然web-0, web-1重调度了，但是PV总是会挂载到合适的挂载点上。
【注】还是因为没有PV，但是通过hostPath+nodeSelector可以达到同样的结果。

### Scaling a StatefulSet

### Updating Containers

### Deleting StatefulSets

暂时没有需求，就不翻了。


---
