---
title: "informer和controller-runtime源码"
date: 2021-11-20T14:17:23+08:00
draft: false
comments: true
toc: true
autoCollapseToc: false
keywords: []
tags: ["kubernetes"]
---
## 问题

{{< figure src="/informer.jpeg" width="500px" >}}

最近在写operator时，碰到一个场景需要让controller每隔1分钟做一次reconcile，而不需要借助外部事件触发。

虽然`client-go`和`controller-runtime`都提供了很方便的接口进行设置。但是知其然还要知其所以然，借着解决这个问题的契机，仔细阅读了一下`infomer`和`controller-runtime`的代码，搞清楚了底层实现原理。

## 方法一

创建`informerFactory`对象时，设置`defaultResync`参数

{{< figure src="/1.png" width=800px" >}}

### 原理

reflector除了会watch apiserver，还会每隔 defaultResync 从indexer中重新获取Object，并将其入队fifo，这样就会重新触发一次informer的Add事件并入队工作队列。

### client-go源码

reflector在进行[ListAndWatch](https://github.com/kubernetes/client-go/blob/10e087ca394e2987f09e759438f9949a746c1ca0/tools/cache/reflector.go#L254)的同时也会周期性的做resync：

{{< figure src="/2.png" width=500px" >}}

这里的store就是fifo，fifo的Resync实现如下：

{{< figure src="/3.png" width=500px" >}}

这里调用knownObjects.ListKeys()来获取所有的Object key然后再入队fifo，这个knownObjects其实就是indexer cache（一个带锁的map）

{{< figure src="/00.png" width=600px" >}}

### controller-runtime源码

对于controller-runtime库，reflector、informer、indexer等组件被封装在cache对象中，cache对象是Manager对象的属性，它们之间的关系如下图所示：

{{< figure src="/controller-runtime.drawio.svg" width=500px" >}}

我们可以通过在创建Manager对象时，传入SyncPeriod参数来达到这一目的，当然SyncPeriod应该是可配置的：

{{< figure src="/4.png" width=700px" >}}

## 方法二

使用`controller-runtime`库时，还可以通过在`Reconcile`中，设置返回的`Result.RequeueAfter`为1分钟来实现：

{{< figure src="/5.png" width=700px" >}}

### 原理

先找到Reconcile的调用点：

1. ControllerManagedBy通过**Builder模式**将Controller添加进Manager中

    {{< figure src="/6.png" width=700px" >}}

2. Manager启动时会启动所有controller，对于controller，“启动”的含义就是启动多个goroutine循环的从workqueue中取key，然后执行Reconcile，顺着Manager.Start一层层的找到[Controller的Start入口](https://github.com/kubernetes-sigs/controller-runtime/blob/master/pkg/internal/controller/controller.go#L148)，最终可以看到熟悉的 `processNextWorkItem`：

    {{< figure src="/7.png" width=700px" >}}

    `processNextWorkItem`的逻辑当然就是从workqueue里取key，然后执行Reconcile的业务逻辑：

    {{< figure src="/8.png" width=700px" >}}

3. 可以看到当`result.RequeueAfter > 0`时，执行了`c.Queue.Forget(obj)`和`c.Queue.AddAfter(req, result.RequeueAfter)`，分别是什么意思呢？要搞清楚这一点，首先我们要弄清楚workqueue的实现。

### workqueue

1. workqueue的创建方法定义在controller中：：

    {{< figure src="/9.png" width=700px" >}}

2. 这里创建的是一个限速队列，限速队列由**延迟队列**和**限速器**两部分组成：

    {{< figure src="/10.png" width=700px" >}}

### AddAfter

`AddAfter`是延迟队列提供的方法，它向`waitingForAddCh`这个channel中传入了一个构造的`waitFor`对象

{{< figure src="/11.png" width=700px" >}}

而这个channel的接收方，则是在创建延迟队列时启动的一个goroutine：

{{< figure src="/12.png" width=700px" >}}

在这个goroutine中，收到`waitFor`对象后，如果还没到执行时间，则会插入优先级队列中（**可以看到，高性能定时器一般用堆实现**）

{{< figure src="/13.png" width=700px" >}}

随后会判断优先级队列中堆顶元素的时间是否到达，如果时间到了，就取出堆顶元素，并入队workqueue，时间没到就计算需要等多长时间，然后启动一个timer进行等待

{{< figure src="/14.png" width=700px" >}}

### Forget

在看Forget方法前，先看限速队列中我们最常用的`AddRateLimited`方法，一般这个方法会在我们Reconcile失败的时候进行调用，目的就是以某种限定的速率重新入队workqueue，从而达到限制重试速度的目的：

{{< figure src="/15.png" width=700px" >}}

可以看到其实就是调用延迟队列的`AddAfter`方法，只是`AddAfter`的方法的参数不是固定的时间，而是由ratelimiter计算得到

[workqueue](https://github.com/kubernetes/client-go/blob/10e087ca394e2987f09e759438f9949a746c1ca0/util/workqueue/default_rate_limiters.go)包中提供的默认限速器是**指数退避限速器 + 令牌桶限速器**：

{{< figure src="/16.png" width=700px" >}}

`Forget`是ratelimiter提供的方法，其实就是把失败的对象从ratelimiter中移除，这样ratelimiter就不会再根据该对象的失败次数对其进行限速计算了

{{< figure src="/17.png" width=700px" >}}

因此，在Reconcile执行成功后，**需要调用Forget将对象（也就是字符串namespace/name）从限速器中移除**，否则会重复入队workqueue一次并且会影响后续限速器对于相同key的限速计算。

回到最开始的问题，当设置`result.RequeueAfter`为1min时，会调用`c.Queue.Forget(obj)`和`c.Queue.AddAfter(req, result.RequeueAfter)`，也就是说1分钟之后会再次入队，触发reconciliation。
