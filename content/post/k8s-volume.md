---
title: "Kubernetes Volume实现原理"
date: 2020-12-12T12:32:33+08:00
draft: false
comments: true
keywords: ["kubernetes", "容器", "Linux"]
tags: ["kubernetes", "容器", "Linux"]
---

## 容器运行时挂载卷的过程

如果CRI是通过dockershim实现的话，kubelet通过CRI接口去拉起一个容器，就好比是通过docker-daemon执行`docker run`命令。

而如果想要在容器中挂载宿主机目录的话，就要带上`-v`参数，以下面这条命令为例：

```shell
docker run -v /home:/test ...
```

它的具体的实现过程如下：

1. 创建容器进程并开启Mount namespace

    ```c
    int pid = clone(main_function, stack_size, CLONE_NEWNS | SIGCHLD, NULL); 
    ```

2. 将宿主机目录挂载到容器进程的目录中来

   ```c
   mount("/home", "/test", "", MS_BIND, NULL)
   ```

    > 此时虽然开启了mount namespace，只代表主机和容器之间mount点隔离开了，容器仍然可以看到主机的文件系统目录。

3. 调用 `pivot_root` 或 `chroot`，改变容器进程的根目录。至此，容器再也看不到宿主机的文件系统目录了。

## kubelet挂载卷的过程

当一个Pod被调度到一个节点上之后，kubelet首先为这个Pod在宿主机上创建一个Volume目录：

**/var/lib/kubelet/pods/<Pod的ID>/volumes/kubernetes.io~<Volume类型>/<Volume名字>**。

在kubernetes中，卷`volumes`是Pod的一个属性，而不是容器的。kubelet先以Pod为单位，在宿主机这个Volume目录中准备好Pod需要的卷。接着启动容器，容器启动时，根据`volumeMounts`的定义将主机的这个目录下的部分卷资源挂载进来。挂载的过程如前所述，相当于为每个容器执行了命令：

```shell
docker run -v /var/lib/kubelet/pods/<Pod的ID>/volumes/kubernetes.io~<Volume类型>/<Volume名字>:/<容器内的目标目录> 我的镜像 ...
```

而kubelet是怎么把卷挂载到主机的volumes目录下的呢？这取决于Volume的类型。

### 远程块存储

1. Attach：将远程磁盘挂载到本地，成为一个主机上的一个块设备，通过`lsblk`命令可以查看到。
   > Attach 这一步，由`kube-controller-manager`中的`Volume Controller`负责

2. Mount：本地有了新的块设备后，先将其格式化为某种文件系统格式后，就可以进行mount操作了。
   > Mount 这一步，由kubelet中的`VolumeManagerReconciler`这个控制循环负责，它是一个独立于kubelet主循环的goroutine。

### NFS

NFS本身已经是一个远程的文件系统了，所以可以直接进行mount，相当于执行：

```shell
mount -t nfs <NFS服务器地址>:/ /var/lib/kubelet/pods/<Pod的ID>/volumes/kubernetes.io~<Volume类型>/<Volume名字> 
```

### hostPath

hostPath类型的挂载方式，和宿主机上的Volume目录没啥关系，就是容器直接挂载指定的宿主机目录。

### emptyDir、downwardAPI、configMap、secret

这几种挂载方式，数据都会随着Pod的消亡而被删除。原因是kubelet在创建Pod的Volume资源时，其实是在主机的Volume目录下创建了一些子目录供容器进行挂载。Pod被删除时，kubelet也会把这个Volume目录删掉，从而这个Volume目录中的子目录也都被删除，这几种类型的数据就被删掉了。

> 远程块存储、NFS存储等持久化的存储，和hostPath、emptyDir、downwardAPI、configMap、secret不一样，**不是在Pod或任何一种workload中的volume字段中直接定义的**，而是在PV中定义的。

## PVC、PV和StorageClass

在Pod中，如果想使用持久化的存储，如上面提到的远程块存储、NFS存储，或是本地块存储（非hostPath），则在volumes字段中，定义`persistentVolumeClaim`，即PVC。

PVC和PV进行绑定的过程，由`Volume Controller`中的`PersistentVolumeController`这个控制循环负责。所谓“绑定”，也就是填写PVC中的`spec.volumeName`字段而已。`PersistentVolumeController`只会将StorageClass相同的PVC和PV绑定起来。

StorageClass主要用来动态分配存储(Dynamic Provisioning)。StorageClass中的`provisioner`字段用于指定使用哪种[存储插件](https://kubernetes.io/docs/concepts/storage/storage-classes/#provisioner)进行动态分配，当然，前提是你要在kubernetes中装好对应的存储插件。`parameters`字段就是生成出来的PV的参数。

> `PersistentVolumeController`只是在找不到对应的PV资源和PVC进行绑定时，借助StorageClass生成了一个PV这个API对象。具体这个PV是怎么成为主机volume目录下的一个子目录的，则是靠前面所述的Attach + Mount两阶段处理后的结果。当然如果是NFS或本地持久化卷，就不需要`Volume Controller`进行Attach操作了。

## 本地持久化卷

对于本地持久化卷，通过在PV模版中

* 定义`spec.nodeAffinity`来指定持久化卷位于哪个宿主机上
* 定义`spec.local.path`来指定宿主机的持久化卷的路径。

此外，由于`PersistentVolumeController`只会将StorageClass相同的PVC和PV绑定起来，所以还需要创建一个StorageClass，并且使PVC和PV中的`StorageClassName`相同。

在 StorageClass 里，进行了如下定义：`volumeBindingMode: WaitForFirstConsumer`，这个字段的作用是**延迟绑定PV和PVC**。定义了这个字段，PVC和PV的绑定就不会在`PersistentVolumeController`中进行，而是由**调度器**在调度Pod的时候，根据Pod中声明的PVC，来决定和哪个PV进行绑定。

本地持久化卷是没办法进行 Dynamic Provisioning的，所以StorageClass字段中的`provisioner`定义的是`kubernetes.io/no-provisioner`。但是它的Static Provisioning也并不需要纯手工操作。运维人员可以使用[local-static-provisioner](https://github.com/kubernetes-sigs/sig-storage-local-static-provisioner)对PV进行自动管理。它的原理是通过DaemonSet检测节点的`/mnt/disks`目录，这个目录下如果存在挂载点，则根据这个路径自动生成对应的PV。所以，运维人员只需要在node节点上，在`/mnt/disks`目录下准备好挂载点即可。

> Q：hostPath可以是挂载在宿主机上的一块磁盘，而不是宿主机的主目录，这种情况使用hostPath作为持久化存储不会导致宿主机宕机。那是不是可以使用hostPath代替PVC/PV作为本地持久化卷？
>
> A：不可以。这种玩法失去了`PersistentVolumeController`对PVC和PV进行自动绑定、解绑的灵活性。也失去了通过`local-static-provisioner`对PV进行自动管理的灵活性。最关键的是失去了**延迟绑定**的特性，调度器进行调度的时候，无法参考节点存储的使用情况。
>
> Q：删除一个被Pod使用中的PVC/PV时，kubectl会卡住，为什么？
>
> A：PVC和PV中定义了`kubernetes.io/pvc-protection`、`kubernetes.io/pv-protection`这个finalizer字段，删除时，资源不会被apiserver立即删除，要等到`volume controller`进行**pre-delete**操作后，将finalizer字段删掉，才会被实际删除。而`volume controller`的**pre-delete**操作实际上就是检查PVC/PV有没有被Pod使用。
