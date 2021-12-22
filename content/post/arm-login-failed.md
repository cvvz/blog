---
title: "【问题定位】串口登录失败"
date: 2019-05-24T02:45:41+08:00
draft: false
comments: true
toc: true
autoCollapseToc: false
keywords: ["Linux"]
tags: ["Linux"]
---

## 问题现象

通过串口无法正常登录ARM设备，shell闪退。

## 问题分析

首先梳理一下SSH登录和串口登录两种方式的流程：

{{< figure src="/linux-login.drawio.svg" width="400px" >}}

1. 两种登录方式首先都要经过PAM插件的处理，SSH登录是由SSHD通过子进程的方式启动shell，串口登录则是拉起/bin/login，由/bin/login启动shell替代自己。
2. shell启动后，会去执行`/etc/profile`中的一系列脚本，配置系统环境。

这里面可能出问题的环节有：PAM插件、/bin/login进程、/bin/bash和/etc/profile。但这个问题的现象是/bin/bash被拉起后，很快又闪退了，因此问题肯定出在/etc/profile脚本中。最后排查发现脚本中限制了串口登录的终端设备名为`ttyS0`，否则直接退出。但新的ARM设备的串行终端名称是`ttyAMA0`。
