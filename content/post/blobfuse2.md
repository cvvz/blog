---
title: "记一次background fuse mount问题的解决过程"
date: 2023-03-18T14:27:04+08:00
draft: false
comments: true
toc: true
autoCollapseToc: false
keywords: []
tags: []
---

> [blobfuse2](https://github.com/Azure/azure-storage-fuse#blobfuse2---a-microsoft-supported-azure-storage-fuse-driver) 是微软开源的使用 golang + libfuse 实现的fuse 文件系统。[blob csi driver](https://github.com/kubernetes-sigs/blob-csi-driver) 是其对应的csi driver实现。我们最近碰到了两个background fuse mount引发的问题，最后定位到了是相同根因并解决：在PR[#1088](https://github.com/Azure/azure-storage-fuse/pull/1088)里分析了问题原因和解决办法，这里也记录一下整个过程。

## 问题现象

我们先后碰到了两个问题，问题现象不同，但是是由同一个根因造成的：

* 问题一： 执行`blobfuse2 mount`挂载实际失败，但没有在终端或者日志里提示任何错误信息，而且errno返回为0： [#1081](https://github.com/Azure/azure-storage-fuse/issues/1081)。这导致blob csi driver将Pod目录挂载到了本地盘。

* 问题二：删除使用了blobfuse2作为挂载点的Pod后，新创建出来的Pod数据丢失：[#1079](https://github.com/Azure/azure-storage-fuse/issues/1079#issuecomment-1462302216)。
    1. 这个问题首先查到的原因是：在删除老Pod时，执行`NodeUnPublishVolume`阶段，unmount bind mountpoint时，会将source mountpoint，也就是blobfuse2的mount point也一并unmount掉。这样就会导致在新Pod创建时，执行`NodePublishVolume`阶段，bind mount的source mountpoint其实是本地文件系统。
    2. 进一步分析时，在创建老Pod阶段，进行`NodePublishVolume`时，目录其实被同时bind mount到了fuse和ext4两个文件系统里：

        {{< figure src="/blobfuse2/1.jpeg" width="1000px" >}}

    3. 而unmount这种mountpoint时就会导致source mountpoint也被unmount掉。虽然不知道这个行为背后的原因是什么，但是问题的根因还是在于bind mount时不应该同时mount到两个不同的文件系统路径。

## 问题原因

libfuse提供了[-f参数](https://github.com/libfuse/libfuse/blob/master/lib/helper.c#L135)来决定是否要让用户态的fuse进程运行在foreground。默认是以background模式运行 —— 即父进程fork出子进程然后退出，子进程被1号进程接管。

blobfuse2是基于golang实现的，通过cgo调用libfuse库函数，但是[golang没法很好的支持fork](https://github.com/golang/go/issues/227)，实际测试时发现如果blobfuse2直接依赖libfuse库中的fork来启动子进程，任何文件系统的读写命令都会被卡住且无法退出。blobfuse2借助了开源项目[go-daemon](https://github.com/sevlyar/go-daemon)来实现在golang中fork。我会在下一篇[博客](https://cvvz.github.io/post/go-daemon/)中分析一下go-daemon的源码。

具体的做法([代码在这](https://github.com/Azure/azure-storage-fuse/blob/5e06d431845e46a2df4bca187a863b71f6e7cb0b/cmd/mount.go#L405-L421))是在start fuse之前，通过go-daemon的`Reborn()`函数fork出子进程，然后子进程[以foreground方式start fuse](https://github.com/Azure/azure-storage-fuse/blob/8f655cf9e1b501c574b1a217bdb57a9e717bb712/component/libfuse/libfuse2_handler.go#L212-L218)：

{{< figure src="/blobfuse2/f.png" width="800px" >}}

并且父进程退出：

{{< figure src="/blobfuse2/fork1.png" width="700px" >}}

这样做会导致问题的原因是，其实[libfuse在daemonize fork之前还有很多逻辑，其中就包括fuse mount](https://github.com/libfuse/libfuse/blob/master/lib/helper.c#L351-L359)：

{{< figure src="/blobfuse2/fusemount.png" width="600px" >}}

也就是说执行daemonize之前很可能发生其他错误导致fuse mount直接报错退出。

{{< figure src="/blobfuse2/beforemount.png" width="600px" >}}

但是现在blobfuse2的fork发生在libfuse的入口点[fuse_main](https://github.com/libfuse/libfuse/blob/master/lib/helper.c#L307)之前了，而且父进程过早的直接退出了，这就导致fuse daemonize之前发生的错误都无法被父进程捕捉到。这也就是为什么执行`blobfuse2 mount`操作即使mount失败了，也不会报错。

第一个问题的原因已经清楚了。

第二个问题的原因也是异步mount导致的：创建Pod时，在`NodeStageVolume`阶段，blob csi driver会调用`blobfuse2 mount`执行fuse mount操作。`blobfuse2 mount`虽然返回成功了，但是实际上mount操作还在子进程中异步的执行。

而csi driver认为`blobfuse2 mount`返回成功了就代表fuse mount成功了，因此也返回执行成功的response。接着kubelet会调用`NodePublishVolume`执行bind mount，但在bind mount时，其实fuse mount仍在进行中，因此首先bind mount到了还未fuse mount成功的ext4文件系统上，等到fuse mount成功，目录被初始化为fuse文件系统，又bind mount到了fuse文件系统上，这就是为什么会同时bind mount到两个文件系统上。

## 调研

因为golang的fork的限制，上述问题只会在使用golang的fuse项目中出现。

[cgofuse](https://github.com/winfsp/cgofuse)也是一个通过cgo+libfuse实现的fuse项目，[只支持foreground fuse mount](https://github.com/winfsp/cgofuse/blob/master/fuse/host.go#L648-L652):

{{< figure src="/blobfuse2/cgofuse.png" width="700px" >}}

blobfuse v1和[s3fs-fuse](https://github.com/s3fs-fuse/s3fs-fuse)都是用c++写的，[直接调用libfuse的fuse_main](https://github.com/s3fs-fuse/s3fs-fuse/blob/master/src/s3fs.cpp#L5574)，没有问题。

## 解决方案 

搞清楚了libfuse底层原理，解决方案似乎很容易想到了，这就是一个父子进程间IPC的场景，但是实际上要完全解决还是要想一些办法的。

第一个要解决的问题是父进程如何知道子进程mount失败。子进程mount失败时是会导致进程退出的，因此只要能捕获子进程退出事件即可。直接在父进程中使用`wait`或者监听`SIGCHLD`信号都可以。

第二个问题是，子进程异常退出时，父进程没有在终端和日志里打印错误信息。[libfuse默认是直接将错误日志输出到stderr](https://github.com/libfuse/libfuse/blob/master/lib/fuse_log.c#L16-L21)，那么我们将子进程的stderr重定向输出到父进程即可。如果是通过直接fork产生的子进程，因为父子进程共享open fd，因此只需要在父进程中创建PIPE，然后在子进程中把stderr重定向到PIPE的write fd，并在父进程从PIPE read fd读就可以了。可是go-daemon其实是先fork，然后exec覆盖子进程，父子进程并不共享open fd，所以子进程创建出来后是拿不到父进程创建出的PIPE fd的。所以得绕一下，最后想到的办法是，先在父进程创建PIPE，然后在创建子进程的时候直接设置子进程attribute，将他的stdout/stderr设置为PIPE write fd。这个功能go-daemon还不支持，顺手做了一下：[https://github.com/sevlyar/go-daemon/pull/90](https://github.com/sevlyar/go-daemon/pull/90)。

现在父进程能感知到子进程mount失败的事件了，但是父进程还需要能感知子进程mount成功的事件才行，这样父进程才可以安全的退出。但是fuse mount成功后是不会以任何形式通知父进程的，除非修改libfuse代码，所以这里碰到了一些困难。在看了libfuse的代码之后，最后想到了如下的解决办法：

libfuse提供了很多[callback](https://github.com/libfuse/libfuse/blob/master/include/fuse.h#L324-L828)钩子函数，涵盖了所有的文件系统命令。简单来说，就是用户在执行`cd`,`ls`,`rm`, `mkdir`等文件系统命令时，kernel会调用相应的callback与用户态fuse进程通信，[blobfuse2也注册了很多对应的callback](https://github.com/Azure/azure-storage-fuse/blob/7d86acbc95871a4e513b0087bdb7b68f23b7d5db/component/libfuse/libfuse_wrapper.h#L60-L119)。libfuse在mount成功后kernel自动执行的第一个callback是[init](https://github.com/libfuse/libfuse/blob/master/include/fuse.h#L608-L617)，用于初始化文件系统。所以最后想到的解决办法就是当用户态的fuse进程在执行`init` callback时，给父进程发送一个`SIGUSR1`的信号，当父进程收到这个信号，就知道fuse mount阶段肯定成功了，可以成功退出了。

