---
title: "解剖进程虚拟内存空间"
date: 2019-06-07T23:14:03+08:00
draft: false
comments: true
keywords: ["虚拟内存"]
tags: ["Linux"]
---

对于**32位 x86 Linux操作系统**，典型的进程地址空间如下图所示：

{{< figure src="/linuxFlexibleAddressSpaceLayout.png" width="750px" >}}

每一个进程运行在各自独立的虚拟内存空间中，从0x00000000到0xFFFFFFFF，共4GB。

进程地址空间从低到高依次是：

- **Text Segment：** 机器指令，只读，一个程序的多个进程共享一个正文段。
  
> 如果进程带有调试信息，可以通过`addr2line` + 正文段地址获得对应的源代码位置。

- **Data Segment：** 具有初值的全局/静态变量。

- **BSS Segment：** 未赋初值的全局/静态变量。

- **Heap：** 堆。堆从低地址向高地址生长。堆区内存在分配过程中可能产生内存碎片：

![内存碎片](/fragmentedHeap.png "内存碎片")

> 申请堆内存的接口是阻塞接口，即可能因为暂时分配不到够大的堆空间导致进程让出CPU。

- **Memory Mapping Segment：** 内存映射区。动态库、mmap、共享内存使用的都是内存映射区。

- **Stack：** 栈。栈从高地址向低地址生长。进程栈空间的总大小可通过
`ulimit -s`查看，默认为8MB。栈中不仅存放着局部变量，**每次函数调用时，参数、返回地址、寄存器值等都会进行压栈。**

- **Kernel space：** 进程地址空间的最高1GB是内核空间。**内核空间被所有进程共享**，但是用户态进程只有通过系统调用陷入内核态才能执行内核态代码。

> 参考文章：[https://manybutfinite.com/post/anatomy-of-a-program-in-memory/](https://manybutfinite.com/post/anatomy-of-a-program-in-memory/)
