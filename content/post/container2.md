---
title: "容器2"
date: 2021-04-23T16:52:20+08:00
draft: false
comments: true
keywords: ["容器", "kubernetes"]
tags: ["容器", "kubernetes"]
toc: true
autoCollapseToc: false
---

## 进程

### 单进程模型

容器中的1号进程对于宿主机而言就是一个普通的进程，它的父进程是runC，runC的父进程是containerd-shim。这个containerd-shim用于管理容器进程，类似于init或者systemd进程的作用(回收僵尸进程)，当进程退出时，containerd-shim会通过runC重新将进程拉起。

容器的“单进程模型”意味着容器进程本身，虽然是1号进程，但是它并不具有通常意义上1号进程，如systemd或init所具有的进程管理能力，比如托管孤儿进程，回收僵尸进程等，它就是一个普通的应用进程。

当然也可以给这个1号进程赋予这种能力，如docker启动容器的时候，加上`--init`参数，起来的容器就强制使用 [tini](https://github.com/krallin/tini) 作为 init 进程了。这种1号进程非应用容器，而是由专门的init进程拉起其他所有应用进程的做法，称为“富容器”（rich container）。富容器的好处是可以把容器当成虚拟机一样对待，方便和经典PaaS体系对接。

云原生提倡使用轻量级容器，因为只有当1号进程就是应用进程本身时，才能准确的向容器运行时暴露进程的实际状况，方便使用kubernetes探针，以及依赖这些探针的周边组件，如service等。

### 信号

缺省状态下，

* C 语言程序里，一个信号 handler 都没有注册；
* Golang 程序里，很多信号都注册了自己的 handler；
* bash 程序里注册了两个 handler，bit 2 和 bit 17，也就是 `SIGINT` 和 `SIGCHLD`。

> 可以通过查看 `/proc/$PID/status`中的`SigCgt` 行来了解哪些信号被捕获了（注册了信号处理函数）。

虽然SIGTERM（15）的默认行为是终止进程，但是当1号进程**没有为SIGTERM注册信号处理函数**时，

* 通过`kubectl exec`进入容器后，通过`kill`命令去优雅终止1号进程，是不会退出的
* 通过宿主机kill $PID，进程也不会退出

此外，**无论什么情况下，在容器中通过`kill -9`尝试强杀1号进程都不可能成功**。

具体原因是，`kill` 命令实际上调用了 `kill()` 这个系统调用，内核尝试将信号发送给1号进程之前，在 [sig_task_ignored](https://github.com/torvalds/linux/blob/master/kernel/signal.c#L88-L89) 函数中对一些特殊情况进行了过滤。

注册了信号处理函数后，1号进程又应该怎样处理 `SIGTERM` 呢？**如果直接退出的话，1号进程会向同 Namespace 中的其他进程都发送一个 `SIGKILL` 信号。这会导致容器中的其他进程没有优雅退出。**

所以 [tini](https://github.com/krallin/tini) 的实现方式是：把除了 `SIGCHILD` 以外的其他所有信号都转发给它的子进程；自己则负责通过 `waitpid` 来回收子进程资源，避免僵尸进程的产生。

> 僵尸进程本质上是一个空的`task_struct`，它所拥有的资源（内存、文件句柄、信号量等）都已经被内核回收了，唯一消耗的资源是pid。
>
> 进程实际退出前的僵尸态是有必要的，它会通过`SIGCHILD`信号告诉父进程自己已经死了，让父进程知道子进程的终止状态，进行相应的处理，比如异常退出重新拉起。
>
> 僵尸进程过多会导致pid被占满，无法再运行新的进程。容器的最大进程数量由/sys/fs/cgroup/pids（pid cgroup）下的 `pids.max` 文件限制。

## CPU

### 使用率

kubernetes中Pod的cpu资源的`request` 和 `limit`字段限制的是cpu的**使用率**。

> top命令可以查看cpu的使用率，100%表示瞬时使用了1个CPU，200%表示2个。这个时间是从怎么来的？**是从proc文件系统里拿到指标计算得来的。**
>
> 进程cpu使用率的具体定义是：（进程用户态和内核态在cpu调度中获得的cpu ticks/ 单个 CPU 产生的总 ticks）*100%
>
> tick：Linux 时钟周期性地（比如1/100秒）产生中断，每次中断都会触发 Linux 内核去做一次进程调度，而这一次中断就是一个 tick。

`limit`意味着最大cpu的使用率能达到多少，这个值是通过cpu cgroup的`cpu.cfs_quota_us`（一个调度周期里这个控制组被允许的运行时间）除以 `cpu.cfs_period_us`（CPU调度周期）得来的；

`request`表示即使当整个节点cpu被完全用满时，我的cpu利用率也能达到这么多，它是通过设置`cpu.shares`（节点上cgroup 可用cpu的相对比例）来实现的。

> 对于系统**各个类型的 CPU 使用率**，则需要读取 /proc/stat 文件，得到瞬时各项 CPU 使用率的 ticks 值，相加得到一个总值，单项值除以总值就是各项 CPU 的使用率。

### 容器资源视图隔离

相比使用虚拟机，使用容器，最大的问题在于**资源视图的隔离**。由于容器没有对/proc，/sys等文件系统进行隔离，因此在容器中使用free、top等命令看到的其实是物理机的数据。此外，应用程序可能会从`/sys/devices/system/cpu/online`中获取cpu的核数，来决定默认线程数，比如`GOMAXPROCS`。

我们可以通过[lxcfs](https://github.com/lxc/lxcfs)来对容器资源视图进行隔离，让容器“表现的”更像一台虚拟机。对于go程序，还可以通过[automaxprocs](https://github.com/uber-go/automaxprocs)这个包来在容器中正确设置`GOMAXPROCS`值。

## 内存

### memory.usage_in_bytes

`malloc()`申请的其实是虚拟内存，容器根据进程的实际物理内存使用值`memory.stat[rss]`是否超过了`memory.limit_in_bytes`，再加上`memory.oom_control`来判断是否进行oom。

> 你可以调整`memory.oom_control`参数，这样即使物理内存已经达到上限了，容器还是不会被cgroup干掉，可是这样的话，**由于申请不到物理内存资源，进程会处于可中断睡眠状态。**

cgroup当中的`memory.usage_in_bytes`实际上是由三部分组成：用户态物理内存(`memory.stat[rss]`) + 内核态内存(`memory.kmem.usage_in_bytes`) + page cache(`memory.stat[cache]`)，即`memory.usage_in_bytes` = `memory.stat[rss]` + `memory.stat[cache]` + `memory.kmem.usage_in_bytes`。

有时候我们发现容器的内存使用量`memory.usage_in_bytes`一直等于`memory.limit_in_bytes`，但是也不会发生OOM，是因为实际上每次以**Buffered IO**的方式读写磁盘时，Linux都会先将数据缓存到page cache当中来加快write/read系统调用的速度，也就是`memory.stat[rss]`值比较高，当进程需要物理内存时，操作系统会自动释放一部分page cache内存给rss内存使用。

### swap

kubelet缺省不能在打开swap的节点上运行。配置`--fail-swap-on=false`，kubelet可以在swap enabled的节点上运行。

rss内存中大部分`没有磁盘文件对应`，这种内存称为匿名内存。swap用于在内存资源紧张时，释放部分匿名内存到磁盘的swap空间。

内核的`/proc/sys/vm/swappiness`参数作用是当系统存在swap空间时，是优先释放page cache还是优先释放匿名内存，即写入swap。

cgroup中的`memory.swappiness`和全局的`/proc/sys/vm/swappiness`作用差不多。**唯一区别是设置`memory.swappiness`为0，可以让这个cgroup控制组里的内存禁止使用swap。**

## 存储

### 容器文件系统

容器文件系统UnionFS，从原理上说，就是多个目录联合挂载到一个目录下，读/写这个目录就相当于读/写了对应目录中的内容。常用的有：aufs（没有合到linux kernel主干）、devicemapper和overlayFS。

以OverlayFS为例， OverlayFS有两层，分别是 lowerdir 和 upperdir。lowerdir 里是容器镜像中的文件，对于容器来说是只读的；upperdir 存放的是容器对文件系统里的所有改动，它是可读写的。lower和upper联合挂载到merged。

### blkio cgroup

磁盘io的两个主要性能指标：

* iops：每秒钟磁盘进行IO的次数
* 吞吐量（带宽）：以MB/s为单位，一次IO读写的数据块越大，吞吐量越大。即**吞吐量 = IOPS * 数据块大小**。

cgroup v1的限制：每一个子系统都是独立的，对于某进程，只能**独立的**在各个cgroup子系统中限制它的资源使用。这样的问题在于，对于buffered I/O，它是先把数据写入page cache，再从page cache刷到磁盘；由于blkio和memory两个子系统相互独立，对于buffered I/O就无法限速了。

Cgroup v2的变化：一个进程属于一个**控制组**，在这个控制组里多个子系统可以**协同运行**。对某个进程，在控制组里同时限制memory + blkio就能对Buffered I/O 作磁盘读写的限速

## 网络

容器 Network Namespace 的网络参数并不是完全从宿主机 Host Namespace 里继承的，也不是完全在新的 Network Namespace 建立的时候重新初始化的。在内核函数 [tcp_sk_init()](https://github.com/torvalds/linux/blob/v5.4/net/ipv4/tcp_ipv4.c#L2631) 里，可以看到 `tcp_keepalive` 的三个参数都是重新初始化的，而 `tcp_congestion_control` 的值则是从 Host Namespace 里复制过来的。

在 Linux 中，管理员可以通过 sysctl 接口修改内核运行时的参数。在 `/proc/sys/` 虚拟文件系统下存放许多内核参数。这些参数涉及了多个内核子系统，比如内核子系统（通常前缀为: kernel.）、网络子系统（通常前缀为: net.）等。通过 `sysctl -a` 可以获取所有内核参数列表。

在kubernetes中，如果对内核网络参数有特殊需求，可以通过 [设置Pod的sysctl参数](https://kubernetes.io/zh/docs/tasks/administer-cluster/sysctl-cluster/#%E8%AE%BE%E7%BD%AE-pod-%E7%9A%84-sysctl-%E5%8F%82%E6%95%B0)，或者在给init container赋予特权，并通过 sysctl 修改内核网络参数。

## 安全

### capability

Linux在kernel 2.2之前，只存在root用户和非root用户之分，在2.2之后，将root用户的特权做了更细粒度的划分，每个特权单元称之为[capability](https://man7.org/linux/man-pages/man7/capabilities.7.html)。`privileged`这个参数的意思就是容器拥有所有capability。容器启动时，缺省只有[15个capabilities](https://github.com/opencontainers/runc/blob/v1.0.0-rc92/libcontainer/SPEC.md#security)。

### user namespace

尽管容器中 root 用户的 Linux capabilities 已经减少了很多，但是在没有 User Namespace 的情况下，容器中 root 用户和宿主机上的 root 用户的 uid 是完全相同的，没有隔离。一旦有软件的漏洞，容器中的 root 用户就可以操控整个宿主机。

为了减少安全风险，业界都是建议在容器中以非 root 用户来运行进程。不过在[kubernetes目前还不支持 User Namespace](https://github.com/kubernetes/enhancements/pull/2101) 的情况下，在容器中使用非 root 用户的话，对 uid 的管理和分配就比较麻烦了。
