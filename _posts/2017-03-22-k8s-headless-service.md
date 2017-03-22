---
layout: post
title: "kubernets: Headless Services"
date: 2017-03-22 00:12:12
author: 伊布
categories: tech
tags: kubernets
cover:  "/assets/instacode.png"
---

我知道Service，但[Headless Services](https://kubernetes.io/docs/user-guide/services/#headless-services) 是用来做什么的呢？

Headless Service也是一种Service，但不同的是会定义spec:clusterIP: None，也就是不需要Cluster IP的Service。


还记得Service的Cluster IP是做什么的吗？对，一个Service可能对应多个EndPoint(Pod)，client访问的是Cluster IP，通过iptables规则转到Real Server，从而达到负载均衡的效果（实现原理请见[这里](http://www.datastart.cn/tech/2017/01/20/k8s-service.html)）。如下：

```
# kubectl get service
NAME                      CLUSTER-IP       EXTERNAL-IP       PORT(S)           AGE
nginx-service             10.107.124.218   192.168.128.158   80/TCP,443/TCP    1d
# kubectl describe  service nginx-service    
Name:                   nginx-service
Namespace:              default
Labels:                 <none>
Selector:               component=nginx
Type:                   ClusterIP
IP:                     10.107.124.218
External IPs:           192.168.128.158
Port:                   nginx-http      80/TCP
Endpoints:              10.244.2.9:80
Port:                   nginx-https     443/TCP
Endpoints:              10.244.2.9:443
Session Affinity:       None
No events.
# nslookup nginx-service.default.svc.cluster.local  10.96.0.10
Server:         10.96.0.10
Address:        10.96.0.10#53

Name:   nginx-service.default.svc.cluster.local
Address: 10.107.124.218
```

虽然service有2个endpoint，但是dns查询时只会返回service的地址。具体client访问的是哪个Real Server，是由iptables来决定的。

那么Headless Service的效果呢？

```
# kubectl get service
NAME                      CLUSTER-IP       EXTERNAL-IP       PORT(S)    AGE
nginx                     None             <none>            80/TCP     1h
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
# nslookup nginx.default.svc.cluster.local 10.96.0.10
Server:         10.96.0.10
Address:        10.96.0.10#53

Name:   nginx.default.svc.cluster.local
Address: 10.244.2.17
Name:   nginx.default.svc.cluster.local
Address: 10.244.2.18
```

dns查询会如实的返回2个真实的endpoint。

所以，顾名思义，Headless Service就是没头的Service。有啥用呢？很简单，有时候client想自己来决定使用哪个Real Server，可以通过查询DNS来获取Real Server的信息。

另外，Headless Services还有一个用处。Headless Service的对应的每一个Endpoints，即每一个Pod，都会有对应的DNS域名；这样Pod之间就可以互相访问。我们还是看上面的这个例子。

```
# kubectl get statefulsets web
NAME      DESIRED   CURRENT   AGE
web       2         2         1h
# kubectl get pods
web-0                       1/1       Running   0          1h
web-1                       1/1       Running   0          1h
# nslookup nginx.default.svc.cluster.local 10.96.0.10
Server:         10.96.0.10
Address:        10.96.0.10#53

Name:   nginx.default.svc.cluster.local
Address: 10.244.2.17
Name:   nginx.default.svc.cluster.local
Address: 10.244.2.18

# nslookup web-1.nginx.default.svc.cluster.local 10.96.0.10
Server:         10.96.0.10
Address:        10.96.0.10#53

Name:   web-1.nginx.default.svc.cluster.local
Address: 10.244.2.18

# nslookup web-0.nginx.default.svc.cluster.local 10.96.0.10
Server:         10.96.0.10
Address:        10.96.0.10#53

Name:   web-0.nginx.default.svc.cluster.local
Address: 10.244.2.17
```

如上，web为我们创建的StatefulSets，对应的pod的域名为web-1， web-2。

完整示例：

```
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
  replicas: 1
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
      nodeSelector:
        node: kube-node3
      volumes:
        - name: www
          hostPath:
            path: /mydir

```




---
