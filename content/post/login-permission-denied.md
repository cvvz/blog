---
title: "【问题定位】/bin/bash无权限导致SSH登录失败"
date: 2019-05-23T02:21:56+08:00
draft: false
comments: true
toc: true
autoCollapseToc: false
keywords: ["Linux"]
tags: ["Linux"]
---

## 问题现象

SSH登录主机失败，提示错误：`/bin/bash: Permission denied`。

## 分析过程

Linux处理SSH远程登录的流程如下：

1. SSHD进程后台监听SSH连接
2. 当有连接到达时，启动一个子进程，并打开一个伪终端设备（pts）
3. 从passwd中获得用户id、组id、shell路径等信息，并为打开的子进程设置进程uid、gid等
4. 通过`exec`系统调用将子进程替换为登录shell（这里是`/bin/bash`），shell的0、1、2文件描述符和伪终端相连
5. 用户通过伪终端和主机通信

很明显这里的错误原因是因为设置了登陆用户的uid、gid的子进程没有`/bin/bash`的执行权限导致的，也就是第4步出错。通过`strace`跟踪sshd进程的系统调用，也印证了这一点：子进程确实是在执行`execve(/bin/bash)`时报`Permission denied`。

继续实验，发现只有非root组内用户登录才会报该错误。因此将一个已登录的root组内用户修改为非root组内用户后，通过`strace`跟踪其执行`/bin/bash`的过程，发现是在`open`某个动态库权限不足。查看该库文件及路径的权限，发现原本应该是`755`权限的`/usr`目录变成了`750`。通过`stat`命令查看该文件夹被修改的时间点，定位到是设备上电脚本中的一个bug。

