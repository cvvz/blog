---
title: "谈谈kubernetes中的etcd"
date: 2022-02-10T08:19:53+08:00
draft: false
comments: true
toc: true
autoCollapseToc: false
keywords: ["kubernetes","etcd"]
tags: ["kubernetes","etcd"]
---

## revision

revision是etcd中资源的全局版本号，可以看作一个全局逻辑时钟。etcd 启动的时候默认revision是 1，对**任何一个key**的增、删、更新操作时都会导致其**全局单调递增**。

etcd中每个kv都对应了两种revision，create_revision和mod_revision：

```bash
# etcdctl get hello -w=json | jq
{
  "header": {
    "cluster_id": 12938807918314689000,
    "member_id": 15640011255034253000,
    "revision": 234503652,
    "raft_term": 42
  },
  "kvs": [
    {
      "key": "aGVsbG8=",
      "create_revision": 165796223,
      "mod_revision": 165796223,
      "version": 1,
      "value": "d29ybGQ="
    }
  ],
  "count": 1
}
```

创建一个kv时，create_revision为当前的全局revision；每当更新或者删除这个key时，mod_revision会被更新为当前的全局revision；读取时，不指定revision，则默认读取最新的数据，否则读取该kv在某个版本号下的快照。

### MVCC和乐观锁

数据库可以通过悲观锁或乐观锁（MVCC）来实现事务的隔离级别。MVCC机制的核心思想是保存kv数据的多个历史版本（revision），etcd中的事务也是基于乐观锁机制实现。

**kubernetes资源的resourceVersion对应的是etcd中kv的mod_revision**。

在controller的reconcile控制循环中，我们常常需要更新workload的spec或者status字段，而如果在同一次reconcile中对同一个资源对象进行了两次更新操作，那么第二次更新会报错：`the object has been modified; please apply your changes to the latest version and try again`，原因就是第一次更新过后，资源对象的resourceVersion被更新为了新的版本，在第二次更新时，由于版本过时导致更新操作返回失败。这就是一个典型的乐观并发场景，需要上层业务逻辑自行重试或者避免写冲突的发生。

### 可靠的watch机制

kubernetes借助resourceVersion实现了可靠的watch机制。client端初次进行list & watch时，list就是一个普通的http get请求；第二步发送一个带有 `resourceVersion=0` 和 `watch=true` query string的http get请求建立watch连接。

此处watch请求中resourceVersion=0的含义就是指client端初次与apiserver建立watch连接，api-server会返回cache中的最新数据，并推送事件。

当client与server之间的连接由于网络原因短暂中断时，client会不断的发送watch请求，并指定resourceVersion，连接恢复时，如果resourceVersion比apiserver中缓存的资源最小版本大，那么apiserver会将由于网络连接中断而遗漏的事件和资源对象发送给客户端；但是如果网络中断时间过长，导致resourceVersion太老，则apiserver会返回410 StatusGone “too old Resource Version”的错误给客户端，客户端处理该错误的方式就是重新list该资源。通过版本号的机制，可以保证不会遗漏事件。

但是这里list也可能出现问题。在1.17版本中，reflector逻辑进行了一次修改：1.17之前reflector在list-watch失败后重试时，会统一使用resourceVersion=0去list，此时apiserver的行为是返回apiserver cache中的所有版本资源对象；但是在1.17版本改成了使用指定的resourceVersion去进行list，此时apiserver的行为是返回cache中大于该版本的资源对象，如果cache中没有大于该版本的资源对象，会等待3s，期望etcd能推送新的事件更新apiserver缓存，如果3s内没有更新，则返回错误“Timeout: Too large resource version”。此时reflector需要处理这个超时错误，但是最初的实现是没有进行处理的。[#92537](https://github.com/kubernetes/kubernetes/pull/92537/files)修复了这个问题，办法就是在reflector中单独处理“Too large resource version”错误，如果上一次watch失败是因为该错误导致，reflector在下次重试list时会指定resourceVersion=""，直接从etcd中获取最新的版本。

当然这么做只是一个简单的bugfix，造成问题的根本原因还是由于apiserver的无状态，多个apiserver内部的cache数据可能不一致，解决cache不一致的问题才是从源头解决。具体的解决方法可以参考[这个KEP](https://github.com/kubernetes/enhancements/pull/1878/files)。

## watch的实现原理

1. client轮询

   这是最容易想到和实现的方法，事实上最初etcd提供的watch机制就是通过客户端轮询来实现的。
2. http1.x 分块传输机制

   http1.x提供了分块传输的机制，apiserver与各个客户端之间的watch机制就是借助分块传输机制的实现的。客户端只需要发送一次get请求，apiserver每当收到etcd推送的新的事件时，就返回一个头部带有`Transfer-Encoding: chunked`的http响应。
3. websocket

   websocket不像http协议里那样是应答-响应式的交互，它在TCP协议之上，通过在应用层进行二进制帧的组包，达到全双工通信的能力。websocket协议和http其实完全不同，但是由于它的主要应用场景是在web服务中，所以它的握手过程搭了http协议的便车：WebSocket 的握手是一个标准的 HTTP GET 请求，但要带上两个协议升级的专用头字段。

   因此通过websocket (ws)也可以实现watch功能。

4. http/2 服务端推送

    etcd v3版本升级到使用基于http/2的gRPC协议。http/2除了解决了TCP连接复用和http协议队头阻塞的问题以外，由于http/2支持服务端主动推送消息，因此etcd可以主动向apiserver push数据，从而相比client端轮询，更加高效的实现了watch机制。

## lease

etcd 的租约模块的使用方法是：

1. client创建lease，并指定TTL
2. client创建kv，并关联某个lease
3. client持续的对lease进行续租

**Kubernetes Event的自动淘汰机制和controller的leader选举都是基于etcd的lease机制实现的**。

controller的选主机制：通过创建锁对象并绑定lease来实现leader续租超时锁被自动销毁；由于所有的client都会watch这个锁的delete事件，从而可以快速发起新的加锁操作。为了避免惊群，client端只会watch比自己revision小的那个key的delete事件，类似于从小到大排队取锁，从而避免所有client同时发起抢锁的情况发生。

etcd是一个基于 Raft 实现的强一致数据库。相比 Redis 基于主备异步复制做数据同步可能导致的锁的安全性问题，etcd中一个写请求需要经过集群多数节点确认，因此一旦分布式锁申请返回给 client 成功后，它一定是持久化到了集群多数节点上，不会出现 Redis 主备异步复制可能导致丢数据的问题，具备更高的安全性。
