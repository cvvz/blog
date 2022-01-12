---
title: "system系统调用探秘"
date: 2019-05-30T00:28:40+08:00
draft: false
comments: true
toc: true
autoCollapseToc: false
keywords: ["Linux"]
tags: ["Linux"]
---

> **6月3日更新**
>
> 新的实验又发现使用`/bin/sh`和书中行为一致，但使用`/bin/bash`的行为和本文中的实验一致，看来是不同shell的底层的实现方式有差异。**而之所以之前用`/bin/sh`做的实验和书中行为不一致，是因为在我做实验的机器上，`/bin/sh`其实是一个指向`/bin/bash`的软链接**。。。
>
> 不过至少得到一个重要结论，那就是：**对于不同的底层shell，system系统调用的表现会不同**。这一点在编码时需要特别注意。

## system实现原理

`system`这个系统调用的源码在网上已经有很多了，这里就不展示了。简单来说，就是父进程`fork`后，在子进程中通过执行`execl("/bin/sh", "sh", "-c", cmdstring, (char *)0)`，使得`/bin/sh`成为新的子进程，然后在`/bin/sh`中执行`cmdstring`命令；父进程循环执行`waitpid`，等待子进程退出的信号。

## 到底有几个子进程？

### 实验一

在学习《UNIX环境高级编程（第3版）》信号一章时，根据图10-27所示，执行 `system("/bin/ed")` 命令后，会分别调用`fork`/`exec`系统调用两次：

1. 第一次发生在调用`system`时，父进程`fork`一次，子进程执行`execl("/bin/sh","sh","-c","/bin/ed",(char *)0)`一次，子进程被替换为`/bin/sh`。
2. 第二次发生在`/bin/sh`这个子进程中，`/bin/sh`会先`fork`一个子进程，这个子进程执行`exec("/bin/ed")`，用`/bin/ed`替换`/bin/sh`。

但是我在自己做实验时，用`strace`命令跟踪系统调用的过程，发现`system`系统调用执行过程中，**只`fork`了一次，`exec`了两次，主要的差异在于`/bin/sh`并没有`fork`子进程，而是直接执行了`exec("/bin/ed")`**。

### 实验二

我在shell下执行`sh -c "sleep 5"&`命令，根据书中的示例，执行`ps -f`后应该可以看到4个进程：

* `ps -f`
* 当前shell进程
* 当前shell的子进程`sh`
* `sh`的子进程`sleep 5`；

**但实际我只看到三个进程，缺少子进程`sh`，`sleep 5`直接成为了当前shell的子进程**：

```shell
Storage:~ # sh -c "sleep 5" &
[1] 101978
Storage:~ # ps -o pid,ppid,cmd
PID PPID CMD
48673 48658 -bash
101978 48673 sleep 5
103012 48673 ps -o pid,ppid,cmd
```

## system的返回值到底是多少？

使用如下程序对system的返回值进行实验：

```c
#include <stdio.h>
#include <stdlib.h>
void main ()
{
    int iStatus;
    iStatus = system("sleep 5");
    if (WIFEXITED(iStatus))
    {
        printf("normal exit code %d ,status %x\n",WEXITSTATUS(iStatus),iStatus);
    }

    if (WIFSIGNALED(iStatus))
    {
        printf("signal code %d ,status %x\n",WTERMSIG(iStatus),iStatus);
    }
}
```

在这个实验程序中，通过`system`系统调用执行`sleep 5`，sleep期间通过`ctrl+C`向main进程发送`SIGINT`信号，观察会打印出什么。

按照书中的实验结果，是会打印"normal exit..."的，原因分析：

1. 收到`SIGINT`信号的是`sleep 5`进程，`sleep 5`进程异常退出时，它的退出值为2；
2. 但`sleep 5`的父进程是`/bin/sh`，而**shell会将子进程的退出码（此处为2）+128作为退出值正常退出（低8位全0）**。
3. 所以父进程`main`通过`waitpid`得到`/bin/sh`的退出码为130，认为是正常执行退出（低8位全0）。

然而我的实验结果却是打印"signal code 2"。原因如下：

实际上`sleep 2`是main函数的子进程，所以，它收到信号退出，main函数通过`waitpid`得到的子进程退出的状态码就是2了。

> **waitpid得到的状态码**：**低7位代表信号值，第8位代表是否core，高8位代表exit退出码。由此可见信号最多127种，exit最大值为255**。
>
> 通过`WIFEXITED`宏判断低七位的信号值是否为0，0为正常退出；通过`WEXITSTATUS`得到高8位的exit值；否则通过`WIFSIGNALED`得到低七位的信号值。
