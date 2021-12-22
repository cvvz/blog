---
title: "kubernetes网络之service"
date: 2020-12-30T15:42:01+08:00
draft: false
comments: true
toc: true
autoCollapseToc: false
keywords: ["kubernetes", "Linux"]
tags: ["kubernetes", "Linux"]
---

在kubernetes中，service其实只是一个保存在etcd里的API对象，并不对应任何具体的实例。service即k8s中的“微服务”，而它的服务注册与发现、健康检查、负载均衡等功能其实是底层watch service、endpoint、pod等资源的DNS、kube-proxy，以及iptables等共同配合实现的。

## 从集群内部访问ClusterIP服务

在[kubernetes网络之DNS
](https://cvvz.github.io/post/k8s-network-dns/)一文中，已经详细说明了从域名到ClusterIP的转换过程。

下面以kubernetes集群中某个Pod访问`kubernetes`服务（kube-apiserver）为例，分析一下kubernetes是怎么将对ClusterIP的访问转变成对某个后端Pod的访问的。

> 注：kube-proxy以iptables模式工作

```shell
➜  ~ k get svc | grep kubernetes
kubernetes                      ClusterIP      192.168.0.1       <none>                  443/TCP                                             348d

➜  ~ k get ep kubernetes
NAME         ENDPOINTS                                                AGE
kubernetes   10.20.126.169:6443,10.28.116.8:6443,10.28.126.199:6443   348d
```

1. 首先数据包从容器中被路由到cni网桥，出现在宿主机网络栈中。
2. Netfilter在`PREROUTING`链中处理该数据包，最终会将其转到`KUBE-SERVICES`链上进行处理：
```shell
-A PREROUTING -m comment --comment "kubernetes service portals" -j KUBE-SERVICES
```
3. `KUBE-SERVICES`链将目的地址为`192.168.0.1`的数据包跳转到`KUBE-SVC-NPX46M4PTMTKRN6Y`链进行处理：
```shell
-A KUBE-SERVICES -d 192.168.0.1/32 -p tcp -m comment --comment "default/kubernetes:https cluster IP" -m tcp --dport 443 -j KUBE-SVC-NPX46M4PTMTKRN6Y
```
4. `KUBE-SVC-NPX46M4PTMTKRN6Y`链以**相等概率**将数据包跳转到`KUBE-SEP-A66XJ5Q22M6AZV5X`、`KUBE-SEP-TYGT5TFZZ2W5DK4V`或`KUBE-SEP-KQD4HGXQYU3ORDNS`链进行处理：
```shell
-A KUBE-SVC-NPX46M4PTMTKRN6Y -m statistic --mode random --probability 0.33332999982 -j KUBE-SEP-A66XJ5Q22M6AZV5X
-A KUBE-SVC-NPX46M4PTMTKRN6Y -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-TYGT5TFZZ2W5DK4V
-A KUBE-SVC-NPX46M4PTMTKRN6Y -j KUBE-SEP-KQD4HGXQYU3ORDNS
```
5. 而这三条链，其实代表了三条 DNAT 规则。DNAT 规则的作用，就是将 IP 包的目的地址和端口，改成 `--to-destination` 所指定的新的目的地址和端口。可以看到，这个目的地址和端口，正是后端 Pod 的 IP 地址和端口。而这一切发生在Netfilter的`PREROUTING`链上，接下来Netfilter就会根据这个目的地址，对数据包进行路由。
```shell
-A KUBE-SEP-A66XJ5Q22M6AZV5X -p tcp -m tcp -j DNAT --to-destination 10.20.126.169:6443
-A KUBE-SEP-TYGT5TFZZ2W5DK4V -p tcp -m tcp -j DNAT --to-destination 10.28.116.8:6443
-A KUBE-SEP-KQD4HGXQYU3ORDNS -p tcp -m tcp -j DNAT --to-destination 10.28.126.199:6443
```
6. 如果目的Pod的IP地址就在本节点，则数据包会被路由回cni网桥，由cni网桥进行转发；如果目的Pod的IP地址在其他节点，则要进行一次容器跨节点通信，跨节点通信的过程可以参考[kubernetes网络之CNI与跨节点通信原理](https://cvvz.github.io/post/k8s-network-cross-host/)这篇文章。

## 从集群外部访问NodePort服务

以下面这个服务(**NodePort为`31849`**)为例：

```shell
➜  ~ k get svc webapp
NAME     TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
webapp   NodePort   192.168.15.113   <none>        8081:31849/TCP   319d
```

1. kube-proxy会在主机上打开31849端口，并配置一系列iptables规则：
```shell
$ sudo lsof -i:31849
COMMAND      PID USER   FD   TYPE     DEVICE SIZE/OFF NODE NAME
kube-prox 253942 root   12u  IPv6 1852002168      0t0  TCP *:31849 (LISTEN)
```
2. 入口链`KUBE-NODEPORTS`是`KUBE-SERVICES`中的**最后一条规则**：
```shell
-A KUBE-SERVICES -m comment --comment "kubernetes service nodeports; NOTE: this must be the last rule in this chain" -m addrtype --dst-type LOCAL -j KUBE-NODEPORTS
```
3. 先跳到`KUBE-MARK-MASQ`链打上**特殊记号`0x4000/0x4000`**，这个特殊记号**后续在`POSTROUTING`链中进行SNAT时用到**。
```shell
-A KUBE-NODEPORTS -p tcp -m comment --comment "default/webapp:" -m tcp --dport 31849 -j KUBE-MARK-MASQ

-A KUBE-MARK-MASQ -j MARK --set-xmark 0x4000/0x4000
```
4. 然后跳到`KUBE-SVC-BL7FHTIPVYJBLWZN`链：
```shell
-A KUBE-NODEPORTS -p tcp -m comment --comment "default/webapp:" -m tcp --dport 31849 -j KUBE-SVC-BL7FHTIPVYJBLWZN
```
5. 后续的处理流程和上一节描述的相同，直到找到了目的Pod IP。
6. 如果目的Pod IP地址就在本节点，则路由给cni网桥转发；如果目的Pod IP在其他节点，则需要进行容器跨节点通信。**注意，这种情形下，本节点相当于网关的角色，在将源数据包转发出去之前，需要进行SNAT，将源数据包的源IP地址，转换为网关（本节点）的IP地址，这样，数据包才可能原路返回，即从目的节点经过本节点返回到实际的k8s集群外部的客户端**：
```shell
-A KUBE-POSTROUTING -m comment --comment "kubernetes service traffic requiring SNAT" -m mark --mark 0x4000/0x4000 -j MASQUERADE
```
这条规则的意思就是：带有`0x4000/0x4000`这个特殊标记的数据包在离开节点之前，在`POSTROUTING`链上进行一次SNAT，即`MASQUERADE`。而这个特殊标记，如前所述，是在外部客户端数据流入节点时打上去的。

## 总结

从上面的分析中，可以看出来，kube-proxy iptables模式中，最重要的是下面这五条链：

* **KUBE-SERVICES**：ClusterIP方式访问的入口链；
* **KUBE-NODEPORTS**：NodePort方式访问的入口链；
* **KUBE-SVC-***：相当于一个负载均衡器，将数据包平均分发给`KUBE-SEP-*`链；
* **KUBE-SEP-***：通过DNAT将Service的目的IP和端口，替换为后端Pod的IP和端口，从而将流量转发到后端Pod。
* **KUBE-POSTROUTING**：通过对路由到其他节点的数据包进行SNAT，使其能够原路返回。

> 对于NodePort类型的service，**如果本节点上没有目的Pod，则本节点起到的是网关的作用**，将数据路由到其他节点。在这种情况下，**访问Pod IP的链路会多一跳**。我们可以通过将`externalTrafficPolicy`字段设置为`local`，当这样本节点上不存在Pod时，`FORWARD`链上的`filter`表规则会直接把包drop掉，而不会从本节点转发出去：
```shell
-A KUBE-NODEPORTS -p tcp -m comment --comment "default/webapp:" -m tcp --dport 31849 -j KUBE-XLB-BL7FHTIPVYJBLWZN

-A KUBE-XLB-BL7FHTIPVYJBLWZN -m comment --comment "default/webapp: has no local endpoints" -j KUBE-MARK-DROP

-A KUBE-MARK-DROP -j MARK --set-xmark 0x8000/0x8000

-A KUBE-FIREWALL -m comment --comment "kubernetes firewall for dropping marked packets" -m mark --mark 0x8000/0x8000 -j DROP
```

## kube-proxy的IPVS模式

上述流程描述的是kube-proxy的iptables模式的工作流程，这个模式最大的问题在于：

* kube-proxy需要为service配置大量的iptables规则，并且刷新这些规则以确保正确性；
* iptables的规则是以链表的形式保存的，对iptables的刷新需要遍历链表

解决办法就是使用IPVS模式的kube-proxy。IPVS是Linux内核实现的四层负载均衡，因此相比于通过配置iptables规则进行“投机取巧”式的负载均衡，IPVS更加专业。IPVS
和iptables一样底层也是基于netfilter，但使用更高效的数据结构（散列表），允许几乎无限的规模扩张。

创建一个service时，IPVS模式kube-proxy会创建一块虚拟网卡，并且把service的ClusterIP绑在网卡上，然后设置这个网卡的后端real server，对应的是EndPoints，并设置负载均衡规则。这样，数据包就会先发送到kube-proxy的虚拟网卡上，然后转发到后端Pod。

IPVS没有SNAT的能力，所以在一些场景下，依然需要依赖iptables。但是使用IPVS模式的kube-proxy，不存在上述两个问题，性能要优于iptables模式。
