---
title: "VolumeBindingMode"
date: 2022-10-16T21:02:59+08:00
draft: false
comments: true
toc: true
autoCollapseToc: false
keywords: ["kubernetes", "csi"]
tags: ["kubernetes", "csi"]
---

[VolumeBindingMode](https://kubernetes.io/docs/concepts/storage/storage-classes/#volume-binding-mode)是storageclass的一个字段，这个字段可以设置为两个值：`Immediate` 或者 `WaitForFirstConsumer`，`WaitForFirstConsumer`的作用有两个：

* static binding时，起到延迟绑定PVC和PV的作用
* dynamically provision时，起到延迟创建PV的作用

具体啥意思，又是怎么实现的？通过走读代码来加深理解。以下涉及到scheduler、persistentvolume controller和external provisioner三个组件的交互，全部都是业务逻辑，所以理解起来比较直白，不用绕弯子，主要任务是把这些业务逻辑walk through一遍。

首先，在集群中创建了Pod和PVC。Pod会触发调度器的调度流程，PVC会触发pv controller的binding流程以及external provisioner的provision流程。下面一个一个来看。

## scheduler

先从调度器说起，重点在[volumebinding插件](https://github.com/kubernetes/kubernetes/blob/master/pkg/scheduler/framework/plugins/volumebinding/volume_binding.go)中。

### Prefilter

在Prefilter阶段，如果有`Immediate`的PVC存在，就必须先与PV绑定完成之后，才允许进行调度。这一点很关键。

{{< figure src="/VolumeBindingMode/prefilter.png" width="1000px" >}}

### Filter

在Filter阶段，对于已经bound的PVC，就必须要Node满足PV的affinity要求。也就是说既要满足Pod中定义的topology，又要满足PV的topology。

{{< figure src="/VolumeBindingMode/filter-0.png" width="1000px" >}}

对于unbound的PVC，先看这个PVC有没有`volume.kubernetes.io/selected-node` annotation，有的话，只能选择annotation指定的node。

{{< figure src="/VolumeBindingMode/filter-1.png" width="1000px" >}}

否则在集群中寻找提前创建的且满足PVC要求的PV，同时Node也要满足PV的affinity要求，进行PV->PVC的绑定（这属于static binding），否则这个PVC需要进行dynamic provision。

{{< figure src="/VolumeBindingMode/filter-2.png" width="1000px" >}}

如果存在需要进行dynamic provision的PVC，就再进行如下过滤，主要是判断：1. 判断Node是否能满足sc里定义的topology 2. CSIStorageCapacity这个feature开启了的话，需要判断节点是否有足够的volume容量

{{< figure src="/VolumeBindingMode/filter-3.png" width="1000px" >}}

### Score

在优选(Score)阶段，只需要根据static binding的情况来进行打分：基于volume capacity的情况来对节点进行优选

{{< figure src="/VolumeBindingMode/score.png" width="1000px" >}}

### Reserve

在预占(Reserve)阶段，对于static binding 的PV，在这里做PV -> PVC的绑定，即更新PV的ClaimRef字段。

{{< figure src="/VolumeBindingMode/reserve.png" width="1000px" >}}

对于dynamic provision的PVC，则设置`volume.kubernetes.io/selected-node` annotation

{{< figure src="/VolumeBindingMode/reserve-2.png" width="1000px" >}}

### PreBind

在PreBind阶段，先向API Server更新PVC和PV。前面在Reserve这个步骤中已经更新了本地缓存，现在是更新到API Server中。即对于static bind的PVC，设置PV的ClaimRef，对于dynamic provision的PVC，设置了`volume.kubernetes.io/selected-node` annotation。

然后等待PV controller完成PVC -> PV的绑定，完成的标志是`pvc.Spec.VolumeName`和`pv.kubernetes.io/bind-completed` annotation被设置

{{< figure src="/VolumeBindingMode/prebind.png" width="1000px" >}}

## persistentvolume controller

这部分的代码位于[pv_controller.go](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/volume/persistentvolume/pv_controller.go)中。

pv controller尝试为PVC寻找一个合适的volume，但是如果是`WaitForFirstConsumer`的PVC，根据上面调度器的分析，调度器会选择合适的PV并进行PV->PVC的绑定，而pv controller则不会做这件事。

{{< figure src="/VolumeBindingMode/volumecontroller.png" width="1000px" >}}

而有了PV -> PVC的绑定，pv controller只需要完成PVC -> PV的绑定并设置`pv.kubernetes.io/bind-completed` annotation 即可。

但是如果是`Immediate`的PVC，根据上面的分析，如果不bound，调度器压根就不会调度Pod。所以是靠controller来找合适的PV，并且完成PV和PVC的绑定。

但是如果集群里没有合适的预先创建好的PV可供PVC进行static binding，那么就要先触发external provisioner进行dynamic provision pv。触发的方式就是设置`volume.kubernetes.io/storage-provisioner`这个annotation的值为指定的provisioner name。

## external provisioner

[external provisioner](https://github.com/kubernetes-csi/external-provisioner)中的ProvisionController负责根据watch到的PVC事件来决定要不要为它provision volume并创建PV。

{{< figure src="/VolumeBindingMode/provisioner-1.png" width="1000px" >}}

在`shouldProvision`中，首先判断`volume.kubernetes.io/storage-provisioner` annotation设置的provisioner name是不是自己；如果是`WaitForFirstConsumer`的PVC，还必须要求`volume.kubernetes.io/selected-node`这个annotation存在，才能进行provision。

注释中还提到，provisioner会通过remove `volume.kubernetes.io/selected-node` 这个annotation来触发调度器来进行reschedule。因为有的时候在指定节点上进行dynamic provision时，还可能因为资源等问题而失败，这在调度阶段并不能感知，所以就有重调度的需求。

{{< figure src="/VolumeBindingMode/provisioner-2.png" width="1000px" >}}
