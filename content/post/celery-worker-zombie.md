---
title: "Celery Worker僵尸进程问题定位记录"
date: 2020-03-30T10:36:36+08:00
draft: false
comments: true
toc: true
autoCollapseToc: false
keywords: ["celery", "Linux"]
tags: ["celery", "Linux"]
---

> 组内有一个基于Flask + rabbitMQ + Celery搭建的web平台，最近在上面开发需求时碰到了一个比较有趣的问题，在这里记录下来。

## 问题背景

web平台整体架构图如下所示：

{{< figure src="/platform.drawio.svg" width="1000px" >}}

Flask向rabbitMQ发送任务消息，后者再将任务分发给不同的Celery worker进行处理。由于每一个任务的处理时间较长，为了不阻塞worker处理下一个任务，在worker中，通过两次fork的方式，生成孤儿进程在后台进行任务处理。

## 问题现象

1. worker 生成的孤儿进程在抛出异常后，没有自动退出，仍然处于运行状态。

2. kill worker的父进程（SIGTERM），父进程不退出，很多worker变成僵尸进程。（**所有的celery worker都是由同一个父进程fork出来的**）

## 排查过程

这个问题基本上是通过走读代码定位出来的，下面给出简化后的worker代码便于后面分析。

```python
def worker():
    while True:
        # 等待任务...
        wait_task()

        try:
            # 处理任务
            execute()
        except:
            on_failure()
        
        ...


def execute():
    try:
        pid = os.fork()
        if pid < 0:
            raise Exception
        elif pid == 0:
            pid = os.fork()
            if pid < 0:
                raise Exception
            elif pid == 0:
                # 实际执行任务处理，遇到异常直接raise
                do_execute()
                os._exit(0)
            else:
                os._exit(0)
        else:
            return
    except:
        raise
```

## 问题原因

在分析问题原因前，先来运行这样一段代码：

```python
if __name__ == '__main__':
    pid = os.fork()
    if pid < 0:
        print "fork failed"
    elif pid == 0:
        print "child pid ", os.getpid()
    else:
        print "parent pid ", os.getpid()
    print "pid ", os.getpid()
```

运行结果是：

```shell
parent pid  78216
pid  78216
child pid  78217
pid  78217
```

从这个结果我们可以看出，**fork出来的子进程虽然和父进程不共享堆栈（子进程获得父进程堆栈的副本），但是他们共享正文段**，所以他们都执行了程序的最后一行，各自输出了自己的pid。

接着来分析上述worker的代码，在`execute()`中，通过两次fork，最终使得`do_execute()`运行在一个孤儿进程中，如果正常运行，最终会执行`os._exit(0)`正常退出。然而，如果运行过程中抛出异常又会发生什么呢？根据父子进程共享正文段这一结论，我们可以知道这个孤儿进程抛出的异常会被第32行的`except`捕获到，并继续向上抛出异常，然后会被第9行`worker()`中的`except`捕获，并执行`on_failure()`。**也就是说，这个孤儿进程最终执行到了worker的代码里去了，而worker本身是一个死循环，因此这个孤儿进程就不会退出了。理论上来说，最终它会运行到第4行，成为一个“worker副本”，等待接收任务**。

至于为什么kill worker的父进程会导致worker变僵尸进程，需要深入研究一下celery源码中的信号处理方法。猜测是父进程在退出前，会先保证所有worker子进程已经退出，而它误以为这个“worker副本”也是自己的子进程，但是却没办法通过向子进程发送信号的方式使其退出，于是就阻塞住了自己的退出流程。而其他已经正常退出的worker就会一直处于僵尸状态。

