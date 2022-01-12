---
title: "浅谈开源集群联邦的设计和实现原理"
date: 2022-01-11T19:47:01+08:00
draft: false
comments: true
toc: true
autoCollapseToc: false
keywords: ["kubernetes"]
tags: ["kubernetes"]
---

## 集群联邦

集群联邦是为了解决单k8s集群节点数量有限的问题而出现的多集群管理方案。更具体一点，集群联邦要解决三类问题：

1. **多k8s集群管理**。这一块对应有社区的[cluster-registry（已归档）](https://github.com/kubernetes-retired/cluster-registry)项目和[cluster-api](https://github.com/kubernetes-sigs/cluster-api)项目
2. **多集群workload管理**。[A Model For Multicluster Workloads (In Kubernetes And Beyond)](https://timewitch.net/post/2020-03-31-multicluster-workloads/) 这篇文章提出了一种多集群workload的模型，现在的集群联邦的应用管理框架基本上都符合该模型。
3. **多集群service（服务暴露和服务发现）**。社区里有 [Multi-Cluster Services API](https://github.com/kubernetes/enhancements/tree/master/keps/sig-multicluster/1645-multi-cluster-services-api) 的KEP，OCM中有相应的实现。

## [KubeFed v2](https://github.com/kubernetes-sigs/kubefed)

{{< figure src="/kubefed.png" width="650px" >}}

**KubeFed v2和其他集群联邦产品的区别是它的“元集群”中也会有各种workload，KubeFed的功能是复制和传播workload到其他k8s集群**。而其他集群联邦产品的元集群则是专注于管理和分发workload到成员集群中，元集群本身没有任何workload。

集群注册和管理：用户通过定义`Cluster Configuration`来声明哪些成员集群需要纳入管控，以及成员集群信息；

workload管理：

* 用户通过创建`Type Configuration`来声明哪些资源类型需要被纳入管控；
* `Federated Type`可以看成是一个**具体**的资源类型对应的Federated版本，其中定义了资源的：`Template`（资源模板）、`Placement`（要被放置在哪几个集群）、`Overrides`（对某个集群的模版的某些指定字段进行覆盖）
  > 需要注意的是`Federated Type`不是由用户生成的。而是由控制器根据用户在`Type Configuration`中定义的被纳入管控的资源类型，将集群中已有的资源复制到`Federated Type`这个CRD的template字段中。
* KubeFed控制器会根据`Federated Type`去动态的创建和启动两种重要的控制器，sync controller（对应图中的`Propagation`）和status controller（对应图中的`Status`），作用分别是往集群里创建和更新资源 和 获取资源集群中的资源当前的实际状态
* 用户通过定义`ReplicaSchedulingPreference（RSP）`这个CRD来声明某个资源在集群范围内的调度规则。RSP Controller（对应图中的`Scheduling`）会根据调度规则以及资源状态来进行调度。所谓的调度，就是去修改`Federated Type`中的`Placement`或`Overrides`。sync controller watch到`Federated Type`的变更，就会去更新成员集群资源。

## [Karmada](https://github.com/karmada-io/karmada)

{{< figure src="/karmada.png" width="450px" >}}

Karmada最大的特色是**整个控制平面是一个部署在k8s中的定制化的k8s（k8s-on-k8s），包括etcd**。底层的这个k8s唯一的作用就是部署karmada的k8s元集群，karmada只对外暴露元集群apiserver的API（图中`karmada apiserver`）。

**Karmada的k8s原集群中的controller组件，通过启动flag限制了很多内置控制器的运行，最典型的例如不启动replicaset/deployment控制器**。而Karmada的controller（图中`Karmada controllers`）则会watch replicaset/deployment等k8s原生资源，这样，用户在创建一个deployment时，实际是karmada controller对其进行reconcile，reconcile的逻辑自然是将workload分发到成员集群中去。

Karmada的支持Push和Pull两种模式分发workload：push模式下由Karmada控制器将应用推送到成员集群，Pull模式下由运行在成员集群侧的Karmada Agent控制器将应用下拉到本地。

{{< figure src="/karmada-resource-relation.png" width="450px" >}}

Karmada借鉴了KubeFed的很多设计思想。Karmada将KubeFed中定义在同一个`Federated Type`中的`Template`、`placement`、`overrides`拆分成了3个单独的对象：原生资源（图中`Resource Template`）、资源传播策略（图中`propagation policy`）、单集群差异化配置策略（图中`override policy`）。

整体工作流程：

1. Karmada控制器会将k8s原生的workload和`propagation policy`进行绑定生成`Resource Binding`
2. karmada scheduler会根据`Resource Binding`中定义的调度策略，来选择具体的集群
3. resourcebinding controller会根据`Resource Binding`生成`Work`，`Work`里面就是具体资源的manifest
4. execution controller负责根据work去成员集群中创建workload；或者成员集群中的agent感知到work事件，创建workload
5. 对于每一个work对应的资源类型，karmada也会去watch成员集群，并同步到元集群中对应的work状态中。

## [OCM](https://github.com/open-cluster-management-io)

{{< figure src="/ocm-arch.png" width="500px" >}}

OCM整体分为三大部分：

* 部署在成员集群中的klusterlet控制器：
  1. 内置registration-agent控制器用于集群的注册、心跳等
  2. 内置work-agent控制器用于watch中央集群中的work，然后在本集群中创建对应的workload
* 中央集群中的Cluster Manager控制器：
  1. 内置registration-controller：用于集群的管理（注册、删除）等
  2. 内置placement-controller：这个控制器用来进行“跨集群调度”workload。
* 自定义插件（add-on）：
  1. 开发者可以利用OCM提供的 Addon framework 来创建他们自己的管理工具或者将其他开源项目集成进来以加强多集群管理能力。图中，OCM原生自带了两个内置的addon的实现：Application Addon用于应用生命周期管理，Policy Addon用于安全策略管理。

集群注册和管理：通过在成员集群中安装agent组件，agent会向中央集群注册自己，即创建`ManagedCluster`对象，中央集群需要进行accept操作。

workload管理：

1. 定义调度策略：首先在中央集群中创建`placement`对象，其中定义了`predicate`和`priority`规则；控制器会自动创建出`PlacementDecision`对象，调度结果会存放在`PlacementDecision`中。
2. 资源部署：需要创建`ManifestWork`对象，这个对象里定义了manifests字段，即资源模板。对应的cluster上的agent watch到了`ManifestWork`，会在自己的集群内创建对应的资源。并且会将状态同步到中央集群中的`ManifestWork` status字段。

**OCM旨在提供一个精简的多集群/多应用管理的内核，不会去定义面向用户的接口部分，而是希望提供下层能力，使得其他面向最终用户的接口实现可以很轻易的集成进来，换句话说OCM不像其他集群联邦产品那样可以开箱即用**。

OCM 和 kubernetes 开源社区结合的比较的密切：

1. 实现了 kubernetes [sig-multicluster](https://github.com/kubernetes/enhancements/tree/master/keps/sig-multicluster) 的多个设计方案，包括 KEP-2149中的Cluster ID、KEP-1645 Multi-Cluster Services API 中关于 clusterset 的概念。
2. 和其他社区开发者共同推动 [Work API](https://github.com/kubernetes-sigs/work-api) 的开发，很多设计来自于[Multi-Cluster Works API
设计文档](https://docs.google.com/document/d/1cWcdB40pGg3KS1eSyb9Q6SIRvWVI8dEjFp9RI0Gk0vg/edit#)

## [Clusternet](https://github.com/clusternet/clusternet)

{{< figure src="/clusternet-apps-concepts.png" width="1000px" >}}

工作原理和前面几个产品大同小异，因为大家要解决的问题差不多。

说几个有特色的点：

1. clusternet-hub 以聚合层api(AA)的形式运行，不依赖额外的存储和端口。**这个设计使得clusternet比较的轻量，既不会像Kubefed v2那样会生成大量Federated Type资源，也不会像Karmada那样需要在k8s集群中完整的部署一个k8s。同时这个设计也使得 clusternet 可以兼容 K8s 原生 API，客户端需要进行很低成本的改造。**
   > **clusternet中AA的作用，就是把k8s原生的api转换为AA中的映射api，例如pod存在apis/shadow/v1alpha1/pods下。所以客户端必须要进行改造，从原来使用k8s原生的api，变为使用clusternet专用的api。这个改造相当轻量，只需要改一行代码。**
2. 支持直接从hub集群中直接访问管理集群，并且用RBAC进行了鉴权管理（是通过hub集群的apiserver进行转发）这样就简化了多集群的资源管理方式，不用登录到子集群中去查看资源。
3. clusternet-agent会建立与父集群的 TCP 全双工的 websocket 通信信道。目前我还没搞懂这么做的好处在哪。

试玩过程中碰到了几个问题：

1. 先通过clusternet部署多集群deployment，然后我去子集群中把deployment删除了，但是通过clusternet显示deployment的状态还是sucess，而且没有重新帮我把deployment创建出来。这个feature现在已经实现了：[#194](https://github.com/clusternet/clusternet/pull/194)
2. 成员集群的deployment的status、event无法和clusternet中的deployment关联起来，还是得通过查看clusternet定义的crd资源或者去成员集群中查看具体的deployment状态。还是和直接在单k8s集群上操作的体验不一致。**这会直接导致我在单k8s集群中写的operator没法直接移植到clusternet中来**。clusternet项目维护者表示后续会着力解决这个问题。
