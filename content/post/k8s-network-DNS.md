---
title: "kubernetes网络之DNS"
date: 2020-12-30T09:41:51+08:00
draft: false
comments: true
toc: true
autoCollapseToc: false
keywords: ["kubernetes", "DNS"]
tags: ["kubernetes", "DNS"]
---

## 默认DNS策略

Pod默认的[dns策略](https://kubernetes.io/zh/docs/concepts/services-networking/dns-pod-service/#pod-s-dns-policy)是 `ClusterFirst`，意思是先通过kubernetes的**权威DNS服务器**（如CoreDNS）直接解析出A记录或CNAME记录；如果解析失败，再根据配置，将其转发给**上游DNS服务器**。以CoreDNS为例，它的配置文件Corefile如下所示：

```shell
➜  ~ kubectl get cm -n kube-system coredns -o yaml
apiVersion: v1
data:
  Corefile: |
    .:53 {
        errors
        health {
           lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }
kind: ConfigMap
...
```

第17行使用[forward插件](https://coredns.io/plugins/forward/)配置了上游域名服务器为主机的`/etc/resolv.conf`中指定的`nameserver`。


## Service和DNS

尽管kubelet在启动容器时，会将同namespace下的Service信息注入到容器的环境变量中：

```shell
➜  ~ kubectl get svc | grep kubernetes
kubernetes                      ClusterIP   192.168.0.1       <none>        443/TCP                                             347d

➜  ~ kubectl exec -it debug-pod -n default -- env | grep KUBERNETES
KUBERNETES_SERVICE_PORT=443
KUBERNETES_PORT=tcp://192.168.0.1:443
KUBERNETES_PORT_443_TCP_ADDR=192.168.0.1
KUBERNETES_PORT_443_TCP_PORT=443
KUBERNETES_PORT_443_TCP_PROTO=tcp
KUBERNETES_PORT_443_TCP=tcp://192.168.0.1:443
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_SERVICE_HOST=192.168.0.1
```

但是通常情况下我们使用DNS域名解析的方式进行服务注册和发现。

Kubernetes中的DNS应用部署好以后，会对外暴露一个服务，集群内的容器可以通过访问该服务的Cluster IP进行域名解析。DNS服务的Cluster IP由Kubelet的`cluster-dns`参数指定。并且在创建Pod时，由Kubelet将DNS Server的信息写入容器的`/etc/resolv.conf`文件中。

查看`resolv.conf`文件的配置：

```shell
➜  ~ k exec -it debug-pod -n default -- cat /etc/resolv.conf
nameserver 192.168.0.2
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

* `nameserver 192.168.0.2`这一行即表示DNS服务的地址（Cluster IP）为`192.168.0.2`。

* `search`这一行表示，如果无法直接解析域名，则会尝试加上`default.svc.cluster.local`, `svc.cluster.local`, `cluster.local`后缀进行域名解析。

  > 其中`default`是namespace，`cluster.local`是默认的集群域名后缀，kubelet也可以通过`--cluster-domain`参数进行配置。

也就是说：

* 同namespace下，可以通过`nslookup` + `kubernetes`解析域名
* 不同namespace下，可以通过`nslookup` + `kubernetes.default`、`kubernetes.default.svc`、`kubernetes.default.svc.cluster.local`解析域名

因为dns服务器会帮你补齐全域名：`kubernetes.default.svc.cluster.local`

> `{svc name}.{svc namespace}.svc.{cluster domain}`就是kubernetes的FQDN格式。

## Headless Service的域名解析

**无论是kube-dns还是CoreDNS，基本原理都是通过watch Service和Pod，生成DNS记录**。常规的ClusterIP类型的Service的域名解析如上所述，DNS服务会返回一个`A记录`，即域名和ClusterIP的对应关系：

```shell
➜  ~ k exec -it debug-pod -n default -- nslookup kubernetes.default
Server:		192.168.0.2
Address:	192.168.0.2#53

Name:	kubernetes.default.svc.cluster.local
Address: 192.168.0.1
```

Headless Service的域名解析稍微复杂一点。

> ClusterIP可以看作是Service的头，而Headless Service，顾名思义也就是指定他的ClusterIP为None的Service。

### 直接解析

当你直接解析它的域名时，返回的是EndPoints中的Pod IP列表：

> 这个EndPoints后端的Pod，不仅可以通过在service中指定selector来选择，也可以自己定义，只要名字和service同名即可。

```shell
➜  ~ k exec -it debug-pod -n default -- nslookup headless
Defaulting container name to debug.
Use 'kubectl describe pod/debug-pod -n default' to see all of the containers in this pod.
Server:		192.168.0.2
Address:	192.168.0.2#53

Name:	headless.default.svc.cluster.local
Address: 1.1.1.1
Name:	headless.default.svc.cluster.local
Address: 2.2.2.2
Name:	headless.default.svc.cluster.local
Address: 3.3.3.3
```

### 给Pod生成A记录

如果**在`Pod.spec`中指定了`hostname`和`subdomain`，并且`subdomain`和headleass service的名字相同**，那么kubernetes DNS会额外给这个Pod的FQDN生成A记录：

```shell
➜  ~ k exec -it debug-pod -n default -- nslookup mywebsite.headless.default.svc.cluster.local
Server:		192.168.0.2
Address:	192.168.0.2#53

Name:	mywebsite.headless.default.svc.cluster.local
Address: 10.189.97.217
```

> Pod的FQDN是：`{hostname}.{subdomain}.{pod namespace}.svc.{cluster domain}`

### ExternalName Service

ExternalName 类型的Service，kubernetes DNS会根据`ExternalName`字段，为其生成**CNAME记录**，在DNS层进行重定向。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external
  namespace: default
spec:
  type: ExternalName
  externalName: my.example.domain.com
```

```shell
➜  ~ k exec -it debug-pod -n default -- nslookup external
Server:		192.168.0.2
Address:	192.168.0.2#53

external.default.svc.cluster.local	canonical name = my.example.domain.com.
Name:	my.example.domain.com
Address: 66.96.162.92
```
