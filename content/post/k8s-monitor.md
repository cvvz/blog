---
title: "浅谈kubernetes监控体系"
date: 2020-11-20T00:24:35+08:00
draft: false
comments: true
keywords: ["kubernetes", "monitor"]
tags: ["kubernetes"]
---

## 监控和指标

### 理解监控

我们可以把监控系统划分为：采集指标、存储、展示和告警四个部分。

存储使用时序数据库TSDB、前端展示使用grafana、告警系统也有多种开源实现。我重点介绍一下和指标采集相关的内容。

### 理解指标

> **我们所采集的指标 (metrics)，追根溯源，要么来自于操作系统，要么来自于应用进程自身**。

在kubernetes中，有三种指标需要被关注，分别来自于：

* kubernetes基础组件。也就是组成kubernetes的应用进程，如api-server、controller-manager、scheduler、kubelet等。
* node节点。也就是组成kubernetes的机器。
* Pod/容器。也就是业务进程的**运行环境**。

基础设施和kubernetes运维人员主要关注前两项指标，保证kubernetes集群的稳定运行。

而业务方开发/运维主要关注[Pod/容器指标](https://github.com/google/cadvisor/blob/master/docs/storage/prometheus.md#prometheus-container-metrics)，这和以往关注[操作系统性能指标](https://book.open-falcon.org/zh_0_2/faq/linux-metrics.html)大不一样。**在云原生时代，业务进程的运行环境从物理机/虚拟机转变为了Pod/容器。可见，Pod/容器就是云原生时代的`不可变基础设施`**。

### 采集容器指标的过程

1. kubelet内置的cAdvisor负责采集容器指标
2. kubelet对外暴露出API
3. Promeheus、Metrics-Server（取代了Heapster）通过这些API采集容器指标

## Prometheus

Prometheus是Kubernetes监控体系的核心。它的[架构](https://prometheus.io/docs/introduction/overview/#architecture)如官网的这幅示意图所示：

{{< figure src="/prometheus.png" width="700px" >}}

从左到右就分别是采集指标、存储、展示和告警这四大模块。我还是只介绍采集指标相关的内容。

### Prometheus是如何采集指标的

1. 直接采集。Prometheus提供了各语言的[lib库](https://prometheus.io/docs/instrumenting/clientlibs/#client-libraries)，使应用能够对外暴露HTTP端口供prometheus拉取指标值。
2. 间接采集。对于无法通过直接引入lib库或改代码的方式接入Prometheus的应用程序和操作系统，则需要借助[exporter](https://prometheus.io/docs/instrumenting/exporters/#third-party-exporters)，代替被监控对象来对 Prometheus 暴露出可以被抓取的 Metrics 信息。

### Prometheus是如何采集Kubernetes的指标的

* kubernetes基础组件：Prometheus是Kubernetes监控体系的核心，所以这些基础组件当然直接使用lib库采集自己的指标并暴露出API。
* node节点：操作系统的性能指标肯定只能借助[node exporter](https://github.com/prometheus/node_exporter#node-exporter)来采集了。
  > **如果node exporter运行在容器里，那么为了让容器里的进程获取到主机上的网络、PID、IPC指标，就需要设置`hostNetwork: true`、`hostPID: true`、`hostIPC: true`，来与主机共用网络、PID、IPC这三个namespace**。
* Pod/容器。如前所述，Prometheus可以通过kubelet(cAdvisor)暴露出来的API采集指标。

## kubernetes HPA

为了automate everything，采集到了性能指标之后，肯定不能只是发送告警，让运维介入，系统应该具备根据指标自动弹性伸缩的能力。**Kubernetes自身具备了水平弹性伸缩的能力，下面介绍和Kubernetes的垂直弹性伸缩(HPA)能力相关的两个内容**。

### Metrics-Server

Metrics-server（heapster的替代品）**从kubelet中**获取Pod的监控指标，并通过[apiserver聚合层](https://kubernetes.io/zh/docs/concepts/extend-kubernetes/api-extension/apiserver-aggregation/)的方式暴露API，API路径为：`/apis/metrics.k8s.io/`，也就是说，当你访问这个api路径时，apiserver会帮你转发到Metrics-server里去处理，而不是自己处理。**这样，Kubernetes中的HPA组件就可以通过访问这个API获得指标来进行垂直扩缩容决策了**。

> `kubectl top`命令也是通过这个API获得监控指标的。

### Custom Metrics

**但是应用程序往往更依赖进程本身的监控指标（如http请求数、消息队列的大小）而不是运行环境的监控指标做决策**。所以只有Metrics-Server暴露出来的API肯定是不够的，因此，Kubernetes提供了另一个API供应用程序暴露自定义监控指标，路径为`/apis/custom.metrics.k8s.io/`。

Custom Metrics的玩法应该是这样的：

1. 应用程序，或者它的exporter暴露出API供Prometheus采集
2. 造一个自定义Metrics-Server，从Prometheus中获取监控数据
3. HPA组件通过访问`/apis/custom.metrics.k8s.io/`进行决策。
