---
title: "k8s storage 生命周期全流程"
date: 2023-08-17T15:13:22+08:00
draft: false
comments: true
toc: true
autoCollapseToc: false
keywords: ["kubernetes"]
tags: ["kubernetes"]
---

> 本篇不涉及代码细节，但是全是对应的业务逻辑代码。。这篇笔记可以作为k8s storage的运维手册，忘记了细节的时候再拿出来重新过一遍。

## 从Pending 到ContainerCreating

### 调度

Pod被创建出来后，调度器开始进行调度，调度时需要判断Pod使用的PVC的状态，PVC对应的storage class的`VolumeBindingMode` 字段，和PVC的`VolumeName`字段：

1. 如果Pod对应的PVC已经和某个PV bound好了，那么调度时，Node除了需要满足Pod的topo要求，还需要满足bound PV的affinity要求。这种情况后续不再讨论。
2. 如果`VolumeBindingMode==Immediate`，或者PVC的`VolumeName`已经设置（static provision）那么就必须等待PVC和PV完成双向绑定才能进行Pod的调度。
3. 如果为`VolumeBindingMode == WaitForFirstConsumer` ，且PVC的`VolumeName`没有设置，那么调度器会为这个Pod完整的走一遍调度流程：
    1. 如果集群里已经有创建好的或者残留的PV满足PVC的要求，调度器会设置满足要求的PV的`claimRef`字段，相当于完成PV → PVC的绑定
    2. 如果没有，调度器则会找到合适的Node设置pvc annotation`"volume.kubernetes.io/selected-node"` 
    
    **注意，此时Pod仍然处于Pending状态，等待PVC和PV完成双向绑定。**
    

### 双向绑定

<aside>
💡 PV和PVC双向绑定，具体指的是在PV中设置`claimRef`字段，和在PVC中设置`volumeName`字段和`pv.kubernetes.io/bind-completed: "yes"`的annotation

</aside>

如果集群里已经有创建好的或者残留的PV满足PVC的要求：

1. 如果`VolumeBindingMode==Immediate` ， 或者PVC的VolumeName已经设置，那么persistentvolumecontroller 会找到满足要求的那个PV完成双向绑定，完成后PVC和PV的status均为Bound。调度器则会继续调度，此时调度时会将PV的affinity考虑在内。
2. 如果为`VolumeBindingMode == WaitForFirstConsumer` ，且PVC的`VolumeName`没有设置，在调度阶段，调度器会设置满足要求的PV的`claimRef`字段，persistentvolumecontroller只可能找到在调度阶段设置了PV `claimRef`字段且等于PVC的，设置PVC的`volumeName`字段和`pv.kubernetes.io/bind-completed: "yes"` annotation。也就是说这种情况下双向绑定是由调度器和persistentvolumecontroller配合完成。

假如集群里没有任何现成的PV可以满足PVC的要求：

1. 如果PVC的`VolumeName`已经设置，persistentvolumecontroller 则什么都不会做。此时PVC会一直`Pending`，Pod也`Pending`，无事发生，直到有人手动创建一个和`VolumeName`同名的PV为止。
2. 如果`VolumeBindingMode==Immediate` ，persistentvolumecontroller 会设置PVC的`volume.kubernetes.io/storage-provisioner` annotation
3. 如果为`VolumeBindingMode == WaitForFirstConsumer` ，也会设置PVC的`volume.kubernetes.io/storage-provisioner` annotation。

### Provision

external-provisioner 发现PVC里有了`volume.kubernetes.io/storage-provisioner` 这个annotation，并且和自己name一致，才可能开始Provision volume。

1. 如果`VolumeBindingMode==Immediate` ，那么可以直接调用csi进行provision
2. 如果`VolumeBindingMode == WaitForFirstConsumer` ，那么还需要有`"volume.kubernetes.io/selected-node"` annotation才能provision，这个是前面调度器设置的。`WaitForFirstConsumer` 是用来支持volume topology 特性的，这里设置的node的topo信息，后续会作为参数传给csi，csi在创建volume时将会满足这个node的topo要求，避免出现跨az的情况。另外如果创建volume失败，external provisioner还可能删除这个annotation，触发调度器重新调度。

provision成功（或者有现成的PV可用）并且双向绑定也做完，Pod就正式被调度到节点上，**此时Pod从`Pending`状态变成了`ContainerCreating`状态**。

kubelet watch到pod事件，缓存到pod信息到podmanager，volumemanager根据收到的pod信息构造出DSW，然后开始进行reconcile。

## 从ContainerCreating到Running

### attach

reconcile流程首先会`waitForVolumeAttach` 。默认attach/detach操作是由attachdetach controller做的，除非显式的设置kubelet参数`enable-controller-attach-detach=false` （kubelet启动时会默认设置为true）。当`enable-controller-attach-detach=true` 时，kubelet会设置一个annotation：

`volumes.kubernetes.io/controller-managed-attach-detach: "true"` ，attachdetach controller 发现有这个annotation时就知道自己应该负责该node上的volume的attach/detach操作。

如果volume plugin是attachable的，并且`enable-controller-attach-detach=false`，那么就是由kubelet进行attach volume：

- 首先判断是否支持multi attach（只有access mode是`ReadWriteMany`或者`ReadOnlyMany`才支持同时attach到多个节点，否则只允许attach到一个节点上），如果不支持，那么同一时刻只会有一个attach操作。
- 对于csi plugin，attach操作就是创建`volumeattachment`，然后一直卡着等到`volumeattachment`的status变为`attached: true`才算attach成功，成功后，才会更新到ASW中。

**值得一提的是，attach/detach还有后面的mount/unmount操作，都是用的operationexecutor框架，实现上是单独起一个goroutine异步处理的，不会卡住任何reconcile流程。**

如果volume plugin不是attachable的或者`enable-controller-attach-detach=true` ，也就是由attachdetach controller负责attach操作，那么会走到`VerifyControllerAttachedVolume`流程中。在这个流程里：

- 首先如果volume不是attachable的，那么直接更新ASW，相当于直接认为attach成功；
- 如果是attachable的，先判断volume是否`ReportedInUse`，如果不是就直接返回；
- 获取节点的`status.VolumesAttached`字段，如果volume在这个字段中，就说明attachdetach  controller的attach操作已经成功了，就可以更新到ASW中。

当由attachdetach controller负责attach操作时：

- 首先会判断是否已经attach了，因为controller本身有缓存，所以是查看ASW而不是像kubelet那样查看node status；
- 接着如果不支持multiattach，会从ASW里查询是否有别的节点attached了这个volume，如果是则不允许再attach，报错返回
- 接着执行attach volume，attach volume操作本身和kubelet是使用的同一个package（内部有些接口实现有些不同，但是大部分一样），上面已经有介绍。
- attach/detach成功后controller会更新node status里的`VolumesAttached`字段，这样volumemanager就可以通过节点上的这个字段来判断某个volume是否已经attach成功。

创建`volumeattachment`之后，被external-attacher watch到后会调用csi进行attach，成功后会更新`volumeattachment`的status为`attached: true` ，如果失败了也会在status里更新错误原因。

前面提到的`ReportedInUse`是这样来的：**kubelet会周期性的调用volume manager的`GetVolumesInUse` 方法来获取所有attachable的并且应该被attach到这个节点上的volume（只要volume在DSW，就应该attach。必须等到volume既不在DSW也不在ASW就会被从node status里删掉。**），更新到node status的`VolumesInUse`字段。更新完了之后，又会调用volume manager的`MarkVolumesAsReportedInUse` 方法，在DSW中进行标注，设置`reportedInUse = true`，表示volume已经更新到 node status 的`VolumesInUse`字段里去了。

`ReportedInUse` 有两个作用：

1. volumemanager在执行`VerifyControllerAttachedVolume`里要先判断是否已经设置了`ReportedInUse` 才会去决定是否应该设置volume为attached。
2. attachdetach controller依赖node status里的 `ReportedInUse`来判断volume是不是已经被kubelet感知到在进行mount操作了，这决定了controller是否可以安全的detach volume，后续也有提到这一点。

### mount device 和mount volume

kubelet等待attach成功，并将volume信息更新到ASW中后，接着进行mount。先mount device，即global mount point，然后mount volume，即将Pod volume bind mount到global mount point。kubelet等待volume mount成功以后会更新**Pod状态从`ContainerCreating`到`Running`**。

## 从Terminating到Pod被彻底删除

删除pod时，pod进入`Terminating`状态，kubelet开始杀掉所有的容器。必须要确保所有容器都已经被杀死，DSWP才会从DSW中删除pod和volume信息，这样就触发reconcile流程进行unmount/unattach。**注意这时Pod仍然处于Terminating状态。**

### unmount volume

第一步是unmount pod volume，并删除vol.data文件。unmount成功后，pod的volume目录就是空的了，pod就可以彻底的从etcd中删除了，这个时候集群里就查询不到这个pod了。如果pod一直卡在`Terminaing`状态，要么是容器删除不掉，要么是unmount一直没有成功，很可能是kernel出bug了。

**pod被彻底删除以后，只代表unmount volume成功了。unmount device和detach volume还会在后台继续进行。**

## Pod被彻底删除以后

### unmount device

unmount device以后，节点上就完全不存在任何mount point了。

### detach volume

如果不需要unmount device，或者unmount device 成功之后，volumemanager开始进行detach volume。

如果plugin不是attachable的，或者是由controller负责attach/detach，就直接把volume信息从ASW里删掉了。**注意这一步会触发kubelet更新node status中的`ReportedInUse` ，将volume从`ReportedInUse` 中删除掉。这意味着从现在开始attachdetach controller可以开始安全的执行detach操作了。**

如果是由kubelet负责attach/detach，kubelet就执行detach volume操作。对于csi，detach就是删除`volumeattachment`，然后等待`volumeattachment`被彻底从etcd中删除掉，才算detach成功。由于`volumeattachment`中定义了finalizer，所以不会直接被删除，需要等到external-attacher调用csi执行detach并成功，才会被彻底从集群中删除，`volumeattachment`被彻底删除了，才算是detach成功。

如果是controller负责attach/detach，controller进行detach 的前提有两个：

1. 是volume在ASW中存在而在DSW中不存在（这里提到的ASW和DSW指的是atachdetach controller的，不是kublet的），只有**当Pod在集群中被彻底删掉了，DSWP才会将volume从DSW中删除，controller才能开始reconcile；**
2. **detach前还需要确保volume已经被从节点unmounted了才能进行**。controller怎么知道volume已经被unmounted成功了呢？当节点上的`ReportedInUse` 字段被增加或者删除时，controller就会相应的设置ASW中volume的`MountedByNode`字段，这个字段就代表着controller是否可以安全detach。又如前面描述的，只有当kubelet unmount已经成功，彻底从volumemanager的ASW中删除后，才会触发更新node status，将volume从节点的`ReportedInUse` 里删除掉。

detach的流程是：

- 首先从ASW中删除该volume；
- 然后更新node status 的`VolumesAttached`，将volume从中去掉；
- 然后执行detach volume，csi 的 detach volume实现和上面kubelet是同一个，已经说过了。
- 如果detach失败了，会重新把volume加回到ASW中。
- 接下来的`UpdateNodeStatuese` 函数又会把volume也重新加回到node status的`VolumesAttached` 。

另外，除了unmount成功后controller会detach volume以外，还有一些情况，即使`ReportedInUse`仍然存在，也就是说volume没有完成unmount也会进行detach volume（但是仍然要保证Pod已经被彻底删除掉了）：

1. 节点状态不健康，并且已经等待了一个超时时间`maxWaitForUnmountDuration`。
2. 节点被打上了`node.kubernetes.io/out-of-service`污点（**节点被打上**`node.kubernetes.io/out-of-service`**污点后，会force delete掉那个节点上不能容忍该污点的pod**）