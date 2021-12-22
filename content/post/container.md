---
title: "容器"
date: 2020-12-24T10:33:00+08:00
draft: false
comments: true
keywords: ["容器", "kubernetes"]
tags: ["容器", "kubernetes"]
toc: true
autoCollapseToc: false
---

## 容器镜像

容器镜像就是容器的rootfs。通过 Dockerfile 制作容器镜像时，就相当于增加 rootfs 层。通过容器镜像运行一个容器时，操作系统内核先将镜像中的每一层**联合挂载**在一个统一的目录下，然后再通过`chroot`把容器的根目录挂载到这个统一的目录下。

通过 Dockerfile 生成容器镜像时，每个原语执行后，都会生成一个对应的镜像层。需要注意的是，即使原语本身并没有明显地修改文件的操作（比如，ENV 原语），它对应的层也会存在。只不过在外界看来，**这个层是空的**。

Docker 中最常用的联合文件系统（`UnionFS`）有三种：`AUFS`、`Devicemapper` 和 `OverlayFS`。

> overlay2 文件系统最多支持 128 个层数叠加，换句话说 Dockerfile 最多只能写 128 行。

## namespace

通过查看宿主机上的 `/proc/${pid}/ns` 目录可以知道容器进程当前的namespace。同一个Pod下的容器，共享哪些namespace呢？看一眼就知道了：

{{< figure src="/namespace.png" width="650px" >}}

可以看出：

* 不共享的namespace是：mnt（挂载点）、pid（进程号）和uts（主机名）
* 共享的namespace是：ipc（进程间通信）、net（网络）和user（用户）。

我用 ` kubectl exec -it ${pod} -c ${container} -n ${ns} -- sh` 命令运行的sh进程，它的namespace和我指定的`${container}`容器一模一样。`kubectl exec` 本质上是通过`setns`系统调用加入了指定进程的namespace。

{{< figure src="/exec-namespace.png" width="650px" >}}

## cgroups

### cpu cgroup

* cpu.cfs_period_us：CFS（Completely Fair Scheduler）调度算法的一个调度周期
* cpu.cfs_quota_us：CFS 调度算法中，在一个调度周期里这个控制组被允许的运行时间
* cpu.shares：这个值决定了 CPU Cgroup 下控制组可用 CPU 的相对比例。**不过只有当系统上 CPU 完全被占满的时候，这个比例才会在各个控制组间起作用**。

    > `cpu.cfs_quota_us` / `cpu.cfs_period_us` 的值就限制了容器进程的最大cpu使用率。
    >
    > 在操作系统里，`cpu.cfs_period_us` 的值一般是个固定值，所以在kubernetes中，当你设置了Pod的`limits.cpu`的值后，kubelet会去修改cgroup中的`cpu.cfs_quota_us`这个参数来调整容器cpu的使用上限。
    >
    > 在kubernetes中，当设置了 Pod的`requests.cpu` 的值时，kubelet会去调整 `cpu.shares` 这个参数，来保证即使节点cpu使用率被打满了，容器仍然能分得一定量的cpu时间。

### cpu 使用率

cpu时间的使用类型如下图所示：

{{< figure src="/cpu-usage.jpeg" width="650px" >}}

有两种情形可以认为进程处于R（运行态）：

* 在运行队列中，等待cpu调度
* 获得了cpu资源，正在进行cpu运算

进程处于睡眠态（在cpu调度器的等待队列中）也有两种情形：

* 可中断，显示为 S 状态，可能是因为**申请不到资源**导致被挂起
* 不可中断睡眠，显示为 D 状态，可能是因为**等待I/O操作完成**，为了保证数据的一致性，这时进程不响应任何信号

对于进程的 CPU 使用率，只包含两部分:

* 一个是用户态， us 和 ni；
* 还有一部分是内核态，也就是 sy。

至于 wa、hi、si，这些 I/O 或者中断相关的 CPU 使用，CPU Cgroup 不会去做限制。因为本身这些也不属于某个进程的cpu时间。

### cpu 负载

cpu 使用率和 cpu 平均负载的区别：

* cpu使用率是进程使用cpu的时间，包括用户态和内核态的时间之和。
* cpu平均负载≈CPU可运行队列中的进程数+**CPU休眠队列中不可中断状态的进程数**。

当节点上处于D状态的进程数量变多的时候，cpu的平均负载会升高，此时大量进程排队竞争disk I/O资源，但cpu可运行队列中的进程数却很少，所以虽然使用率很低，但是仍然会拖慢进程速度。

### cpu cgroup

cpu cgroup能限制cpu的使用率，但是cpu cgroup并没有办法解决平均负载升高的问题。

我们可以做的是，在生产环境中监控容器的宿主机节点里 D 状态的进程数量，然后对 D 状态进程数目异常的节点进行分析，比如磁盘硬件出现问题引起 D 状态进程数目增加，这时就需要更换硬盘。

### cpuset cgroup

cpuset cgroup用于进程绑核，主要通过设置`cpuset.cpus`和`cpuset.mems`两个字段来实现。

在kubernetes中，当 Pod 属于 Guaranteed QoS 类型，并且 requests 值与 limits 被设置为同一个相等的**整数值**就相当于声明Pod中的容器要进行绑核。

### memory cgroup

* memory.limit_in_bytes：一个控制组里所有进程可使用内存的最大值。一旦达到了这个值，可能会触发OOM。
  > 在kubernetes中，当你指定了 Pod 的 `limits.memory=128Mi` 之后，相当于将 memory cgroup 中的 `memory.limit_in_bytes` 字段 设置为 128 * 1024 * 1024
* memory.usage_in_bytes：当前控制组里所有进程实际使用的内存总和，**包括rss和page cache两部分**。
* memory.oom_control：决定了内存使用达到上限时，会不会触发OOM Killer。触发OOM时，会选择控制组里的某个进程杀掉。
* memory.stat：显示了各种内存类型的实际开销。**其中"cache"代表page cache；"rss"代表进程真正申请到的物理内存大小。RSS 内存和 Page Cache 内存的和，等于`memory.usage_in_bytes` 的值**。判断容器真实的内存使用量，我们不能用`memory.usage_in_bytes`，而需要用 `memory.stat` 里的 rss 值。
* memory.swappiness：定义Page Cache 内存和匿名内存释放的比例。

> Q：当执行 `kubectl exec` 时，创建的进程会加入到容器的cgroup控制组吗？
>
> A：会。以cpu cgroup为例，查看`/sys/fs/cgroup/cpu/kubepods.slice/kubepods-pod{$uid}.slice/docker-{$containerID}.scope/tasks`文件就能发现新创建的进程被加入到容器的cgroup控制组了。
>
> Q：执行 `kubectl top` 命令获取到的pod指标是从哪里来的？
>
> A：整个执行路径是：`kubectl -> apiserver -> aggregated-apiserver -> metric-server -> kubelet(cAdvisor) -> cgroup`。最终来源就是cgroup。而Linux `top`命令的指标数据的来源是`/proc`文件系统。

## kubelet、Docker、CRI、OCI

docker 架构图如下图所示：

{{< figure src="/docker.png" width="800px" >}}

kubelet和docker的集成方案：

{{< figure src="/kubelet-docker.png" width="800px" >}}

从这两幅图就能看出来，当前在kubernetes中，创建一个容器的调用链为：

`kubelet -> dockershim -> docker daemon -> containerd -> containerd-shim -> runc -> container`

dockershim实现了[CRI](https://github.com/kubernetes/kubernetes/blob/8327e433590f9e867b1e31a4dc32316685695729/pkg/kubelet/apis/cri/services.go)定义的gRPC接口，实现方式就是充当docker daemon的客户端，向docker daemon发送命令。实际上dockershim和docker daemon都可以被干掉，[kubernetes在v1.20也的确这么做了](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.20.md#deprecation)。docker从kubernetes中被移除后，我们可以直接使用[containerd](https://github.com/containerd/containerd)或[CRI-O](https://github.com/cri-o/cri-o)作为CRI。

[runC](https://github.com/opencontainers/runc)则是一个[OCI](https://github.com/opencontainers/runtime-spec)的参考实现，底层通过Linux系统调用为容器设置 namespaces 和 cgroups, 挂载 rootfs。当然kubernetes其实不关心OCI的底层是怎么实现的，只要能保证遵循OCI文档里的标准，就能自己实现一个OCI。[Kata](https://github.com/kata-containers/kata-containers)就是遵循了OCI标准实现的安全容器。它的底层是用虚拟机实现的资源强隔离，而不是namespace。

Kata中的VM可以和Pod做一个类比：

* kubelet调用CRI的`RunPodSandbox`接口时，如果是runC实现的OCI，则会去创建`infra`容器，并执行`/pause`将容器挂起；如果是Kata，则会去创建一个虚拟机。
* 接着kubelet调用`CreateContainer`去创建容器，对于runC，就是创建容器进程并将他们的namespace加入`infra`容器中去；对于Kata，则是往VM中添加容器。
