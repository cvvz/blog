---
title: "WSL: VPN的网络问题"
date: 2022-08-22T09:23:12+08:00
draft: false
comments: true
toc: true
autoCollapseToc: false
keywords: ["network", "运维"]
tags: ["network", "运维"]
---

最近由于工作原因，不得不将工作机切换到Windows系统，准备装个WSL作为overlay的工作环境。安装、配置都没什么问题，但是发现主机接了VPN以后，虚拟机没法科学上网了。利用周末时间折腾了一下，在这里记录一下调查和解决问题的过程。

## 环境信息

主机：Windows10, OS build: 19044.1889

虚拟机：Ubuntu 20.04

WSL有1和2两个版本，这两个版本的区别是wsl1以桥接方式加入主机网络，wsl2则有自己的虚拟以太网卡和ip地址，通过NAT的方式访问Internet。主机VPN导致的网络问题在wsl1和wsl2上都存在，但是问题原因不一样，这里分开讨论。

## WSL1

### 主机环境

**网络设备**

{{< figure src="/wsl-vpn/wsl1-host-devices.png" width="1000px" >}}

图中Ethernet是我的物理网卡（192.168.31.8），连接小米路由器（192.168.31.1），MSFTVPN是VPN的虚拟出来的一块虚拟网卡（100.64.16.6）。

**路由表**

{{< figure src="/wsl-vpn/wsl1-host-route.png" width="700px" >}}

值得注意的是路由表开头有两条默认路由规则，内核会选择metric较小的那个作为高优先级的网络接口。(数据流和metric的关系就好像电流和电阻的关系:) )这说明主机流量会强制通过VPN虚拟网卡，无论是访问墙内还是墙外网站。这一点在[官方文档](https://docs.microsoft.com/en-us/windows/security/identity-protection/vpn/vpn-routing#force-tunnel-configuration)中有解释。

### 虚拟机环境

在虚拟机内看到的网络设备和路由表和主机几乎相同。说明桥接模式下，两者共处在同一个网络层下。

{{< figure src="/wsl-vpn/wsl1-vm-devices.png" width="700px" >}}


### 问题和排查过程

问题：执行 `/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"` 试图安装brew发现无响应。

经过排查发现是虚拟机中域名解析出了问题

{{< figure src="/wsl-vpn/wsl1-vm-dns.png" width="700px" >}}

但是在主机中域名解析是OK的。

{{< figure src="/wsl-vpn/wsl1-host-dns.png" width="800px" >}}

修改虚拟机的`/etc/hosts`文件，添加 `185.199.108.133 raw.githubusercontent.com` 可以解决这个问题，但这只是一种workaroud，绕过了域名解析，直接使用IP地址，真正的问题是dns解析失败并没有解决，访问其他网站仍然可能出错。不过这至少说明了虚拟机内部是能够通过VPN访问到Internet的，只要解决了域名的问题，就能解决curl不通的问题了。

那么dns为什么会解析失败？查看配置文件，发现这个wsl自动生成的dns配置文件设置的dns server是小米路由器网关的ip地址192.168.31.1
{{< figure src="/wsl-vpn/dns-config.png" width="900px" >}}

想到在主机上解析域名是成功的，猜测主机使用的dns服务器应该不是路由器网关。由于在主机配置中没有找到其他dns server，所以想抓包看看虚拟机的域名解析和主机有啥区别，结果发现主机上所有与VPN Server通信的数据全部被加密处理了：
{{< figure src="/wsl-vpn/data-capture-host.png" width="900px" >}}

但是进一步证实了虚拟机中域名解析的报文的确是直接发给了网关：
{{< figure src="/wsl-vpn/data-capture-vm.png" width="1000px" >}}

后来又想能不能直接抓VPN的这块虚拟网卡呢，这样不就拿到明文数据了吗？结果发现windows上wireshark看不到这块设备，Ubuntu里系统调用支持不全，抓包失败。。

但是找到了正确的方向解决起来就很快了，在网上查找VPN域名解析相关的内容时找到了这篇[文档](https://docs.microsoft.com/en-us/windows/security/identity-protection/vpn/vpn-name-resolution)，介绍了Windows VPN进行域名解析时，会先去查询[NRPT](https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2008-R2-and-2008/ee649207(v=ws.10))表，我在主机上找到了这张表，其中定义了DNS Server：

{{< figure src="/wsl-vpn/nrpt.png" width="800px" >}}

手动修改虚拟机DNS配置文件，问题解决。

## WSL2

WSL1升级到WSL2之后，这个网络问题仍然存在，但是原因不同。fyi，这个问题我还没找到根因，暂时还是继续用WSL1。。

### 主机环境

WSL2中的虚拟机和主机不在同一个网段，虚拟机通过NAT，转换为主机的ip访问外网（使用VPN时，则是转换为VPN的client ip访问VPN server再访问外网，当然实际流量也是搭载在真实物理网络之上）。升级WSL2后，会在主机上多创建一块虚拟网卡172.20.176.1：

{{< figure src="/wsl-vpn/wsl2-host-devices.png" width="1000px" >}}

### 虚拟机环境

{{< figure src="/wsl-vpn/wsl2-vm-devices.png" width="900px" >}}

可以看到，这时虚拟机有自己的网络设备和路由规则。不再像桥接模式那样与宿主机共用网络设备和路由表。在虚拟机中有一块网卡eth0，ip地址为172.20.179.153/20，默认路由通过这个网卡发到宿主机上的虚拟网卡172.20.176.1。

### 问题排查过程

首先dns解析是没有问题的，ping外网也没有问题，说明整个网络链路是没有问题的。

那么先看看curl命令具体卡在哪一步：

{{< figure src="/wsl-vpn/curl.png" width="900px" >}}

是卡在tls握手流程，没有收到server端的响应。

接着在虚拟机里抓包，但是也只是同样发现client一直没有收到server hello的响应：

{{< figure src="/wsl-vpn/hello.png" width="900px" >}}

物理机上抓包只能看到一堆加密后的数据包，没有任何有帮助的信息。

后来又进一步发现curl大部分https站点有问题，有小部分是没有问题的。但是这也没有给我带来什么实质性的启发。

这个问题我最终没有找到根因。社区里相同的[issue](https://github.com/microsoft/WSL/issues/5068)长达两年了一直在讨论但是一直没有等到ms官方的回应或者修复。。。有一些民间的workaroud: [wsl-vpnkit](https://github.com/sakai135/wsl-vpnkit), [bridge workaroud](https://github.com/microsoft/WSL/issues/4150)，还没有尝试过。

最后如果继续排查这个问题，估计需要借助一些内核工具，沿着数据返回的链路依次调查：宿主机物理网卡->宿主机内核协议栈->解密数据->VPN虚拟网卡->宿主机内核协议栈->DNAT并路由->wsl的虚拟网卡->进入linux内核协议栈。

