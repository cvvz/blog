---
title: "go-daemon 原理解析"
date: 2023-03-19T09:38:40+08:00
draft: false
comments: true
toc: false
autoCollapseToc: false
keywords: []
tags: []
---

> [go-daemon](https://github.com/sevlyar/go-daemon)是golang的[fork问题](https://github.com/golang/go/issues/227)的一种解决方案，在解决[blobfuse2 mount问题](https://cvvz.github.io/post/blobfuse2/)时看了一遍go-daemon源码，在这里记录一下。

go-daemon通过[Reborn](https://github.com/sevlyar/go-daemon/blob/master/daemon.go#L30)函数来模拟fork

{{< figure src="/go-daemon/reborn.png" width="500px" >}}

返回值child为nil时执行子进程的代码，不为nil时执行父进程的代码：

{{< figure src="/go-daemon/child.jpeg" width="600px" >}}

父子进程运行的是同一套代码，在底层是通过env来区分当前执行父进程还是子进程的代码块。也就是说父进程在启动子进程的时候会设置一个特殊的env，然后通过`os.StartProcess`传入env并启动子进程执行当前的二进制文件：

{{< figure src="/go-daemon/startchild.png" width="550px" >}}

`os.StartProcess`底层调用的是`forkExec`，即fork + exec，用新进程覆盖fork出来的子进程。因此父子进程除了执行同一份代码以外，不共享任何资源，只能通过在启动子进程时主动设置一些attributes来实现部分资源共享。

比如go-daemon就是在父进程中打开PIPE，然后设置子进程的stdin为PIPE的read端fd

{{< figure src="/go-daemon/file.png" width="400px" >}}

父进程向PIPE的write端写入数据：

{{< figure src="/go-daemon/write.png" width="400px" >}}

子进程启动时从stdin读取数据并解析

{{< figure src="/go-daemon/read.png" width="400px" >}}

之后把fd 3（在启动子进程时设置为了`/dev/null`）dup到stdin，相当于关闭了stdin

{{< figure src="/go-daemon/dup.png" width="400px" >}}