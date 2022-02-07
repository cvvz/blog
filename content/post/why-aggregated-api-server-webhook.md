---
title: "以aggregated apiserver的方式部署admission webhook"
date: 2020-12-01T07:19:06+08:00
draft: false
comments: true
keywords: ["安全","kubernetes"]
tags: ["kubernetes", "安全"]
toc: true
autoCollapseToc: false
---

## 热身概念：apiserver认证客户端的方式

apiserver为客户端提供三种认证方式：

1. https**双向**认证（注意是双向认证，例如kubeconfig文件中既要配置客户端证书和私钥，又要配置CA证书）
2. http token认证（例如serviceaccount对应的secret中，包含token文件、ca证书，容器就是通过这两个文件和apiserver进行http token认证的）
3. http base认证（用户名+密码）

## admission webhook和扩展apiserver

> 对于admission webhook和扩展apiserver而言，**apiserver可以简单的看作是客户端**，admission webhook和扩展apiserver则作为服务端。

### apiserver通过HTTPS连接admission webhook

当apiserver作为客户端连接admission webhook时，要求admission webhook必须提供https安全认证，但是默认是**单向**认证即可。**也就是admission webhook负责提供服务端证书供apiserver进行验证，但webhook默认可以不验证apiserver**。apiserver所需要的CA证书在webhookconfiguration文件中的`caBundle`字段中进行配置。如果不配置，则默认使用apiserver自己的CA证书。

管理服务端证书的方式有三种：

* 我们可以自己签发webhook的证书，如 `istio` 项目中使用的[脚本](https://github.com/istio/istio/blob/release-0.7/install/kubernetes/webhook-create-signed-cert.sh)
* 像`openkruise`项目一样[在controller中自动生成证书](https://github.com/openkruise/kruise/blob/master/pkg/webhook/util/controller/webhook_controller.go#L262)
* 使用开源的[cert-manager](https://github.com/cert-manager/cert-manager)自动生成和管理证书

### admission webhook验证apiserver

如果你的admission webhook想要验证客户端（也就是apiserver），那么就需要额外给apiserver提供一个配置文件，这个配置文件的内容和kubeconfig很像，可以指定apisever使用http base认证、http token或者证书来向webhook提供身份证明，具体过程详见[官方文档](https://kubernetes.io/zh/docs/reference/access-authn-authz/extensible-admission-controllers/#authenticate-apiservers)。

简单来讲就是：在启动 apiserver时，通过`--admission-control-config-file` 这个参数指定了客户端认证的配置文件，这个文件的格式和用 kubectl 连接 apiserver 时用到的 `kubeconfig` 格式几乎一样。只不过这里，客户端是apiserver，服务端是admission webhook。

通过这种方式验证客户端，最麻烦的地方是需要手工维护kubeconfig，且对于每个webhook都需要维护一个。

### apiserver通过HTTPS（双向）连接扩展apiserver

aggregated apiserver在设计之初就解决了客户端认证的问题，具体实现过程详见[官方文档](https://kubernetes.io/zh/docs/tasks/extend-kubernetes/configure-aggregation-layer/#kubernetes-apiserver-%E5%AE%A2%E6%88%B7%E7%AB%AF%E8%AE%A4%E8%AF%81)。

## 以扩展apiserver的方式部署admission webhook

openshift 的 [generic-admission-server库](https://github.com/openshift/generic-admission-server#generic-admission-server) 是一种用来编写admission webhook的lib库，使用它可以**避免人工创建和维护客户端key和证书的复杂性（即上面提到的apiserver的kubeconfig文件）。不过我觉得它更大的好处是同时也省去了创建和维护服务端证书的步骤，使得开发人员可以更加专注于webhook的功能本身，安全性相关的功能借由kubernetes的聚合层apiserver的双向认证机制自动完成**，这是通过常规方式部署admission webhook时所办不到的。我们来看下它的原理是怎样的：

1. 首先在 WebhookConfiguration 中，配置admission webhook为kubernetes服务，即apiserver自己，并设置path为一个特殊的group、version，**这一步相当于是apiserver把请求转发给了自己**。

    ```yaml
    webhooks:
    - clientConfig:
        service:
          name: kubernetes
          namespace: default
          path: /apis/{group}/{version}/{resource}
          port: 443
    ```

2. 在 ApiService 中，配置聚合api的group、version，以及后端的service：

    ```yaml
    spec:
      group: {group}
      service:
        name: {webhook-service}
        namespace: {namespace}
        port: 443
      version: {version}
    ```

这种方式部署的admission webhook的整个工作流程如下图所示：
{{< figure src="/webhook.drawio.svg" width="400px" >}}

1. apiserver过滤指定的请求，**将它发到自己的路径下**。

2. 再由aggregator转发到扩展apiserver，也就是真正的admission webhook进行处理。

**使用聚合apiserver还有一个额外的好处：可以设置apiservice中的 `spec.insecureSkipTLSVerify`字段为true，使用不安全的连接，这样在测试环境调试时任何证书都不需要了**
