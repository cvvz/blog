---
title: "kubernetes网络之CNI与跨节点通信原理"
date: 2020-12-30T09:51:44+08:00
draft: false
comments: true
toc: true
autoCollapseToc: false
keywords: ["kubernetes","CNI","容器"]
tags: ["kubernetes","CNI","容器"]
---

## 初始化infra容器网络环境

当kubelet通过[RunPodSandbox](https://github.com/kubernetes/kubernetes/blob/master/staging/src/k8s.io/cri-api/pkg/apis/services.go#L66)创建好`PodSandbox`，即infra容器后，就需要调用[SetUpPod](https://github.com/kubernetes/kubernetes/blob/master/pkg/kubelet/dockershim/network/plugins.go#L73)方法为Pod（infra容器）创建网络环境，底层是调用CNI的[AddNetwork](https://github.com/containernetworking/cni/blob/master/libcni/api.go#L80)为infra容器配置网络环境。

这个配置网络环境的过程，就是kubelet从cni配置文件目录（`--cni-conf-dir`参数指定）中读取文件，并使用该文件中的CNI配置配置infra网络。kubelet根据配置文件，需要使用CNI插件二进制文件（存放在`--cni-bin-dir`参数指定的目录下）实际配置infra网络。

这些 CNI 的基础可执行文件，按照功能可以分为三类：

1. **Main 插件**，它是用来创建具体网络设备的二进制文件，比如bridge（网桥设备）、loopback（lo 设备）、ptp（Veth Pair 设备）等等
2. **IPAM（IP Address Management）插件**，用来给容器分配IP地址，比如dhcp和host-local。
3. **CNI 社区维护的内置 CNI 插件**，比如flannel，提供跨主机通信方案

初始化一个容器网络环境的过程大致如下：

1. 没有网桥就使用`bridge`创建一个网桥设备
2. 使用`ptp`创建一个veth pair设备，并且把一端插在容器里，成为容器的eth0网卡，另一段插在网桥上
3. 使用`dhcp`或`host-local`为eth0网卡分配IP地址
4. 调用第三方CNI插件，比如`flannel`，实现容器跨主机通信方案

## 容器跨节点通信

在[浅谈单机容器网络](https://cvvz.github.io/post/container-network/)一文中，已经详细分析了同一主机内部容器之间通过veth + 网桥的方式通信的过程，下面分析一下容器跨主机通信的过程。

容器的跨主机网络方案可以分为两类：**overlay**和**underlay**。

### underlay和overlay

所谓underlay，也就是没有在宿主机网络上的虚拟层，容器和宿主机处于同一个网络层面上。

> 在这种情形下，Kubernetes 内外网络是互通的，运行在kubernetes中的容器可以很方便的和公司内部已有的非云原生基础设施进行联动，比如DNS、负载均衡、配置中心等，而不需要借助kubernetes内部的DNS、ingress和service做服务发现和负载均衡。

所谓overlay，其实就是在容器的IP包外面附加额外的数据包头，然后**整体作为宿主机网络报文中的数据进行传输**。容器的IP包加上额外的数据包头就用于跨主机的容器之间通信，**容器网络就相当于覆盖(overlay)在宿主机网络上的一层虚拟网络**。如下图所示：

{{< figure src="/overlay-network.png" width="600px" >}}

### Flannel UDP模式
  
  Flannel的UDP模式的工作流程：

  1. container-1根据默认路由规则，将IP包发往cni网桥，出现在宿主机的网络栈上；
  2. flanneld预先在宿主机上创建好了路由规则，数据包到达cni网桥后，随即被转发给了flannel0
  3. flannel0的功能就是将数据包传给用户态的flanneld进程
  4. flanneld进程查询etcd，找到目的容器ip地址和目的宿主机ip的对应关系，然后将原ip包封装在一个udp包中发送到目的宿主机上的flanneld进程。
  5. 目的宿主机的flanneld收到包后，反向处理一遍就发送到了目的容器中。
  
  整个过程如下图所示：

  {{< figure src="/flannel-udp.jpg" width="600px" >}}

  由于这中间数据从flannel0发送到了用户态的flanneld，又从flanneld发送到宿主机的eth0网卡，用户态和内核态发生了两次数据传递，且在用户态还进行了封包操作，所以udp模式性能很差。
  
### Flannel VXLAN模式

  Flannel VXLAN模式的原理和UDP模式差不多，区别在于：
  
  1. UDP模式创建的是TUN设备(flannel0)，VXLAN模式创建的是VTEP设备（flannel.1）。
  2. VTEP设备全程工作在内核态，性能比UDP模式更好。
  
  VXLAN模式的工作流程：

  1. container-1根据默认路由规则，将IP包发往cni网桥，出现在宿主机的网络栈上；
  2. flanneld预先在宿主机上创建好了路由规则，数据包到达cni网桥后，随即被转发给了flannel.1，flannel.1是一个VTEP设备，**它既有 IP 地址，也有 MAC 地址**；
  3. **在node2上的目的VTEP设备启动时，node1上的flanneld会将目的VTEP设备的IP地址和MAC地址分别写到node1上的路由表和ARP缓存表中**。
  4. 因此，node1上的flannel.1通过查询路由表，知道要发往目的容器，需要经过10.1.16.0这个网关。**其实这个网关，就是目的VTEP设备的ip地址**。
  ```shell
  $ route -n
  Kernel IP routing table
  Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
  ...
  10.1.16.0       10.1.16.0       255.255.255.0   UG    0      0        0 flannel.1
  ```
  5. 又由于**这个网关的MAC地址，事先已经被flanneld写到了ARP缓存表中**，所以内核直接把目的VTEP设备的MAC地址封装到链路层的帧头即可：
  {{< figure src="/flannel-vxlan-frame.jpg" width="500px">}}
  6. **flanneld还负责维护FDB（转发数据库）中的信息**，查询FDB，就可以通过这个目的VTEP设备的MAC地址找到宿主机Node2的ip地址。
  7. 有了目的IP地址，接下来进行一次常规的、宿主机网络上的封包即可。

  整个过程如下图所示：

  {{< figure src="/flannel-vxlan.jpg" width="600px" >}}

  可以看出，VXLAN模式中，flanneld维护的都是内核态数据：路由表、arp缓存表、FDB，VXLAN模式几乎全程运行在内核态。性能要比UDP模式好不少。
