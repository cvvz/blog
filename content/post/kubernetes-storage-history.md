---
title: "k8s存储演进过程"
date: 2022-06-15T13:22:45+08:00
draft: false
comments: true
toc: true
autoCollapseToc: false
keywords: ["kubernetes"]
tags: ["kubernetes"]
---

最近在做一些CSI相关的工作，重新看了一下之前的笔记和资料，发现kubernetes从最初简单的Volume到现在复杂的CSI的设计，有一个逐步演进的过程，搞清楚一个技术的历史可以帮助我们更好的理解和掌握它。

## Volume

要在一个 Pod 里声明 Volume，只要在 Pod 里加上 `spec.volumes` 字段即可。然后，你就可以在这个字段里定义一个具体类型的 Volume 了，比如：hostPath，emptyDir等。现在通常直接用Volume的情况局限在使用宿主机的本地存储，为什么不推荐通过Volume使用某个具体的网络存储呢？可以看下面这个例子：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: rbd
spec:
  containers:
    - image: kubernetes/pause
      name: rbd-rw
      volumeMounts:
      - name: rbdpd
        mountPath: /mnt/rbd
  volumes:
    - name: rbdpd
      rbd:
        monitors:
          - '10.16.154.78:6789'
        pool: kube
        image: foo
        fsType: ext4
        readOnly: true
        user: admin
        keyring: /etc/ceph/keyring
        imageformat: "2"
        imagefeatures: "layering"
```

可以发现，直接通过Volume使用网络存储有两个问题：

1. 开发者需要熟悉所使用的存储的各种配置参数
2. 暴露了存储服务api、用户名、授权文件等敏感信息

## PVC/PV

为了解决上述两个问题，k8s项目引入了PVC/PV。

PVC面向开发人员，开发人员不用再知道大量的存储实现细节，只需要声明需要的存储容量和读写权限等：

```yml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pv-claim
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

而PV面向存储管理人员，他们熟知存储使用细节：

```yml
kind: PersistentVolume
apiVersion: v1
metadata:
  name: pv-volume
  labels:
    type: local
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  rbd:
    monitors:
    - '10.16.154.78:6789'
    pool: kube
    image: foo
    fsType: ext4
    readOnly: true
    user: admin
    keyring: /etc/ceph/keyring
```

通过PVC/PV解决了开发者使用存储的困难，但是没有解决运维人员管理存储的困难。k8s虽然把网络存储**attach**、**mount**到宿主机和mount到容器的流程自动化了（参考[最后一节](https://cvvz.github.io/post/kubernetes-storage-history/#volume的实现原理)），但是创建（**provision**）网络存储的工作还没有自动化，运维人员还是需要手动创建网络存储和PV。

## StorageClass

为了解决上述问题，引入了StorageClass，借助storageclass和[external-storage库](https://github.com/kubernetes-retired/external-storage)，可以使得存储的provision变得自动化（即自动的创建网络存储和PV）。比如声明下面这个sc：

```yml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: block-service
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
```

则可以借助` kubernetes.io/gce-pd`存储插件（基于[external-storage库](https://github.com/kubernetes-retired/external-storage)开发）自动创建网络存储和PV。

## FlexVolume

现在看起来似乎没什么问题了。但还是有问题，随着各种云存储层出不穷，越来越多的存储厂商想要把自己的存储插件塞到k8s的主干代码（in-tree）中（pkg/volume）。所以k8s想提供一种抽象层，使得新增的存储插件不必和k8s主干一起演进和测试。随后就引入了FlexVolume这种Volume类型。

对于`attach`和`Mount`这两个操作，controller实际上是根据不同的存储类型，调用pkg/volume目录下的存储插件(Volume Plugin)代码，而对于FlexVolume这个Volume类型，就是对应 pkg/volume/flexvolume 这个目录里的代码。

但是这个目录和其他存储插件不一样，它只充当插件的入口，而没有复杂的业务逻辑。这个目录里的代码非常简单，比如mount操作，就是去调用宿主机上的二进制文件，所以当你编写完了 FlexVolume 的实现之后，一定要把它的可执行文件（比如 blobfuse）放在每个节点的插件目录下（`/usr/libexec/kubernetes/kubelet-plugins/volume/exec`）。

## CSI

FlexVolume是一种out-of-tree的解决方案，但是依然不够完美。主要体现在它需要宿主机的权限并在宿主机上安装二进制文件（mount操作需要在worker node上安装二进制文件，attach操作需要在master node上安装二进制文件）

此外，在StorageClass这一节提到我们可以借助[external-storage库](https://github.com/kubernetes-retired/external-storage)来编写存储插件，实现dynamic provision的能力，但是要专门去写一个还是有点麻烦。

为了解决这些问题（以及其他类似问题），社区又提出了CSI方案，彻底把存储插件的管理逻辑和k8s主干代码解耦开来：

1. 不需要节点权限，不需要在节点上安装可执行文件
2. 把公共能力（动态provision、attach等）从k8s主干分支中抽离出来，放在[kubernetes-csi](https://github.com/kubernetes-csi)这个项目中

此外值得注意的是：`The Container Storage Interface (CSI) is a standard for exposing arbitrary block and file storage systems to containerized workloads on Container Orchestration Systems (COs) like Kubernetes.`也就是说CSI是一个标准，除了k8s以外还可以兼容其他容器编排平台，只要按照这个标准进行实现即可，参考[CSI spec](https://github.com/container-storage-interface/spec/blob/master/spec.md)。

CSI本身的运行机制不是本篇的重点，可以参考[kubernetes CSI官方文档](https://kubernetes-csi.github.io/docs/introduction.html)和[设计文档](https://github.com/kubernetes/design-proposals-archive/blob/main/storage/container-storage-interface.md)。比如在设计文档里已经把CSI Driver各个组件的交互过程写的非常清楚了：[Example Walkthrough
](https://github.com/kubernetes/design-proposals-archive/blob/main/storage/container-storage-interface.md#example-walkthrough)，无需赘述。

## Volume的实现原理

这个主题其实和本文没什么关系，放在最后作参考用。

### 本地存储

* emptyDir：直接用 `/var/lib/kubelet/pods/<Pod的ID>/volumes/kubernetes.io~<Volume类型>/<Volume名字>` 目录，所以emptyDir Volume的存储介质（比如Disk还是SSD）由kubelet根目录（一般是/var/lib/kubelet）所在的文件系统决定

* hostPath：通过bind mount的方式把node上的某个路径mount到`/var/lib/kubelet/pods/<Pod的ID>/volumes/kubernetes.io~<Volume类型>/<Volume名字>`

### 网络存储

除了NFS只需要mount操作：`mount -t nfs <NFS服务器地址>:/ /var/lib/kubelet/pods/<Pod的ID>/volumes/kubernetes.io~<Volume类型>/<Volume名字> ` 以外，其他存储（块、对象）都需要两步：

* **attach**: 把远程磁盘attach到宿主机，成为宿主机的一个块设备，比如`gcloud compute instances attach-disk <虚拟机名字> --disk <远程磁盘名字>`

* **mount**: 把块设备格式化成文件系统（NFS不需要），并mount到宿主机上，比如：`mount -t nfs <NFS服务器地址>:/ /var/lib/kubelet/pods/<Pod的ID>/volumes/kubernetes.io~<Volume类型>/<Volume名字> `

kubelet 在向 Docker 发起 CRI 请求时，要先准备好宿主机上的`/var/lib/kubelet/pods/<Pod的ID>/volumes/kubernetes.io~<Volume类型>/<Volume名字>`这个目录，接着通过`docker run -v /var/lib/kubelet/pods/<Pod的ID>/volumes/kubernetes.io~<Volume类型>/<Volume名字>:/<容器内的目标目录> 我的镜像 …` 就把Volume挂载进了容器。
