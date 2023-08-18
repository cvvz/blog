---
title: "记一次通过read-ahead优化NFS性能的过程"
date: 2022-10-16T22:58:42+08:00
draft: false
comments: true
toc: true
autoCollapseToc: false
keywords: ["NFS", "fuse", "read-ahead"]
tags: ["存储", "Linux"]
---

## 问题

agent node操作系统版本Ubuntu 18.04，客户使用[blob csi driver](https://github.com/kubernetes-sigs/blob-csi-driver)，使用NFS协议进行挂载，对一个25GB的文件执行sha256sum耗时20多分钟，性能远低于使用本地磁盘。

## 分析

通过NFS协议挂载和直接挂载本地磁盘的区别是下层的文件系统IO变成了网络IO。

查看sha256sum执行过程中的cpu利用率，发现IO wait偏高，也就是说造成耗时的主要原因是因为执行了过多的网络IO。

在排除了网络延迟的对这个问题的影响后，优化重点放在能否减少网络IO次数上，这和减少本地文件系统IO的优化思路没什么区别。sha256sum是典型的顺序读场景（通过strace可以看到它在不停的执行系统调用`read`，每次读取32KB数据进行处理），因此可以通过增加预读（read-ahead）的数据量来增加每次IO额外顺序读取的数据并缓存，从而增加系统调用读page cache的次数，并减少磁盘IO的次数。

执行 `echo 16384 > /sys/class/bdi/$(mountpoint -d  $MOUNT_POINT)/read_ahead_kb`设置预读参数为16384（默认为128）后，sha256sum的执行时间提高到3分钟。

下面对比一下优化前和优化后的各项指标。

### cpu利用率

#### before

{{< figure src="/nfs-read-ahead/cpu-utilization-0.png" width="700px" >}}

#### after

{{< figure src="/nfs-read-ahead/cpu-utilization-1.png" width="700px" >}}

优化后cpu利用率提升，IO wait时间减少。

### cache命中率

测试工具[cachetop](https://github.com/iovisor/bcc/blob/master/tools/cachetop.py)

#### before

{{< figure src="/nfs-read-ahead/cache-hit-0.png" width="700px" >}}

#### after

{{< figure src="/nfs-read-ahead/cache-hit-1.png" width="700px" >}}

优化后命中率差不多，但每一秒hit/miss的次数更多，也就意味着单位时间内read执行的次数更多，通过cache读取的数据也更多。（每次命中cache读取一个page，4KB大小）

### vfs_read函数执行时间

测试工具[funcinterval](https://github.com/iovisor/bcc/blob/master/tools/funcinterval.py)

#### before

{{< figure src="/nfs-read-ahead/vfs-read-0.png" width="700px" >}}

#### after

{{< figure src="/nfs-read-ahead/vfs-read-1.png" width="700px" >}}

耗时长（需要进行IO）的vfs_read次数减少了，耗时短（读缓存）的vfs_read次数增加了

## 在blobfuse上的发现

实验时还发现使用[blobfuse](https://github.com/Azure/azure-storage-fuse)挂载时，即使不设置`read-ahead`参数，性能也不错，sha256sum的执行时间大约在5分钟左右。

通过strace观察sha256sum系统调用执行情况，发现在执行`openat`时花了很长时间，这期间blobfuse cpu利用率很高（主要时间花在用户态、系统态和处理软中断），io wait也很高，**可以推测sha256sum在执行`openat`的时候，blobfuse就已经在读取数据了**。

{{< figure src="/nfs-read-ahead/blobfuse-0.png" width="700px" >}}

等到执行read的时候，io wait不是很高，sha256sum的cpu利用率很高，说明读取数据并没有经过IO，应该是直接读的page cache。

{{< figure src="/nfs-read-ahead/blobfuse-1.png" width="700px" >}}
