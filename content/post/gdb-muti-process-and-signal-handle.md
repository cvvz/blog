---
title: "gdb中的多线程和信号处理"
date: 2019-06-10T11:44:52+08:00
draft: false
comments: true
keywords: ["gdb"]
tags: ["工具"]
---

## 多线程调试

使用GDB调试多线程时，控制程序的执行模式主要分两种：all-stop 模式和 non-stop 模式。

### All-Stop

>任何一个线程在断点处hang住时，所有其他线程也会hang住。默认为all-stop模式。

1. 在all-stop模式中，当一个线程到达断点或产生信号，GDB将自动选择该线程作为当前线程并停住（提示`Switching to Thread n`），并且其他线程也都会停止运行；
2. 当执行`continue`、`until`、`finish`、`next`、`step`等使线程继续运行，所有线程会同时继续运行，直到某一个线程再次被stop，然后该线程成为当前线程。
3. 这里还存在这样一种情况：当你单步跟踪某个线程时，这个线程一定是执行了某条完整语句后在下一条语句前停住，**但是这段时间里其他线程可能执行了半条、一条或多条语句**。
4. 在all-stop模式下，可以通过设定`scheduler-locking`（调度器锁定）来控制CPU调度器的行为从而控制多线程的并发运行行为。

   - `set scheduler-locking off`：默认调度器锁定为关，也就是CPU也可以进行自由调度，那么所有线程是“同进同止”的，一起stop，一起继续运行，竞争CPU资源；
   - `set scheduler-locking on`：开启调度器锁定，不允许CPU自由调度，CPU只能执行当前线程中的指令，其他线程一直处于stop状态；

### Non-Stop

> 任何一个线程被stop甚至单步调试时，其他线程可以自由运行。

1. 通过`set non-stop on`手动开启non-stop模式。一般non-stop模式搭配异步执行命令使用。
2. GDB的可执行命令分为两种：同步执行和异步执行。
   - 同步执行：即执行一条命令后，要等待有线程被stop了才会在弹出命令提示符。这是默认执行模式。
   - 异步执行：立刻返回弹出命令提示符。打开命令异步执行模式开关的命令是`set target-async on`。
   > 在命令后跟`&`表示该命令以异步的方式执行，如`attach&`、`continue&`等。
3. non-stop模式下可使用`interrupt`停止当前运行中的线程，`interrupt -a`停下所有线程。

## 信号处理

GDB能够检测到程序中产生的信号，并进行针对性的处理。通过`info handle`查看对所有信号的处理方式：

- Stop：检测到信号是否停住程序的运行；
- Print：是否打印收到该信号的信息；
- Pass to program：是否把该信号传给进程处理（或者说是否屏蔽该信号，无法屏蔽`SIGKILL`和`SIGSTOP`信号）

通过`handle SIG`来指定某个信号的处理方式。
