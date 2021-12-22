---
title: "抓包解读smtp和tls协议"
date: 2019-06-22T23:21:54+08:00
draft: false
comments: true
keywords: ["tls"]
tags: ["安全"]
toc: true
autoCollapseToc: false
---

> 背景：某进程调用 `libcurl` 提供的 `curl_easy_perform` 接口与邮箱服务器进行smtp通信时，服务端返回56(`CURLE_RECV_ERROR`)错误。由于服务端日志信息不足，于是想到可以通过抓包查看建立smtp连接时的错误信息。

### 第一次抓包

{{< figure src="/smtp-with-tls.png" width="1050px" >}}

从图中可以清晰看出整个SMTP连接从建立到断开的全过程：

1. 通过三次握手建立TCP连接
2. 客户端向服务端发送 `STARTTLS`，服务端回复 `220 Ready to start TLS`后，SMTP协议准备建立安全信道
3. [TLS协议握手](https://cvvz.github.io/post/about-computer-security/#ssl%E5%8D%8F%E8%AE%AE)建立连接
4. TLS协议建立连接后，**应用层协议的内容就被加密了，抓包只能看到图中的`Application Data`字样**。
5. 通过TCP四次挥手断开连接

> 由于smtp协议内容被加密了，因此需要先去掉TLS连接，再抓包分析。

### 第二次抓包

{{< figure src="/smtp-without-tls.png" width="1050px" >}}

从第二次抓包得到的信息，可以看出连接断开的根因是smtp服务器返回了`502 VRFY disallowed`。

接下来网上搜索`smtp VRFY disallowed`相关内容就能找到答案了：原来`libcurl`从7.34.0版本开始，要求SMTP客户端显式的设置 `CURLOPT_UPLOAD` 选项，否则libcurl将发送`VRFY`命令。而一般服务器出于安全性的考虑，会禁止执行VRFY命令。（参考[https://issues.dlang.org/show_bug.cgi?id=13042](https://issues.dlang.org/show_bug.cgi?id=13042) ）

> 通过抓包还证实了，不进行加密通信的应用层数据是明文传输的，smtp协议中的用户名密码被一览无余。
