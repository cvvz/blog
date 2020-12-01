---
title: "以aggregated API server的方式部署admission webhook"
date: 2020-12-01T07:19:06+08:00
draft: false
comments: true
keywords: ["安全","kubernetes"]
tags: ["kubernetes", "安全"]
---

> openshift 的 [generic-admission-server库](https://github.com/openshift/generic-admission-server#generic-admission-server) 是用来编写admission webhook的lib库，它声称**使用该库可以避免为每一个webhook创建和维护客户端证书和密钥所带来的复杂性，开发者只需要维护服务端密钥和证书即可**。我们来看下它是如何实现的。

首先需要知道的是，由于webhook可以从api server接收API对象并对其进行修改，功能十分强大，因此在生产环境中，webhook和api server之间需要进行双向安全认证。即，客户端（api server）和服务端（webhook）双方都需要提供证书，对方则使用CA证书对证书进行校验。
> 在一次加密通信中，证书、私钥、CA证书是怎么工作的可以参考我之前写的[这篇文章](https://cvvz.github.io/post/about-computer-security/)。
> 
> 以下讨论的关注点在于如何简化**客户端证书**的部署，**即api server向webhook提供的证书这一部分**。webhook向api server提供的服务端证书仍然是需要手工部署的。

## admission webhook认证api server的过程

在启动 api server时，通过`--admission-control-config-file` 这个参数指定了客户端证书、私钥的配置文件，这个文件的格式如下所示：

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: ValidatingAdmissionWebhook
  configuration:
    apiVersion: apiserver.config.k8s.io/v1
    kind: WebhookAdmissionConfiguration
    kubeConfigFile: "<path-to-kubeconfig-file>"
- name: MutatingAdmissionWebhook
  configuration:
    apiVersion: apiserver.config.k8s.io/v1
    kind: WebhookAdmissionConfiguration
    kubeConfigFile: "<path-to-kubeconfig-file>"
```

其中，`kubeConfigFile` 参数指定了 `kubeconfig` 文件的存放位置。`kubeconfig` 文件和用 kubectl 连接 api server 时用到的那个 `kubeconfig` 格式一样。只不过这里，客户端是api server，而不是kubectl。

可以看到，通过这种方式部署的webhook，需要手工管理客户端凭证（kubeConfig文件），且每个webhook都需要生成一个客户端凭证。而在webhook中，还需要使用生成证书所用的CA证书来校验api server。非常麻烦。

## aggregated API server认证api server的过程

aggregated API server 作为api server的另一种服务端，它所实现的校验客户端（api server）的机制相比 admission webhook就更加成熟和容易维护了。

在启动 api server 时，我们只需要指定如下几个参数：

- `--proxy-client-key-file`：客户端私钥
- `--proxy-client-cert-file`：客户端证书
- `--requestheader-client-ca-file`：CA证书

aggregated API server 认证 api server 的过程就是自动进行的：

1. 首先，api server会提前为我们在 kube-system 命名空间中创建一个名为 `extension-apiserver-authentication`的 configmap。这个configmap中存储的正是CA证书。

2. api server 和 aggregated API server 通信时，会发送前面指定的客户端证书，并用私钥进行解密。**而 aggregated API server 用来校验证书的CA证书，就是从第一步生成的configmap中获取的。**

可以看到，这种维护客户端凭证的方式，不需要我们手工维护配置文件和CA证书，我们只需要在启动API server时配置一次，后续API server和aggregated API server会自动获取各自需要的文件。

## 以aggregated API server的方式部署admission webhook

理解了上述两种认证过程，就不难理解为什么以aggregated API server的方式部署webhook可以简化客户端证书的使用了。

部署admission webhook的步骤是：

1. 在(Validating/Mutating)WebhookConfiguration中，配置admission webhook为kubernetes服务：

    ```yaml
    webhooks:
    - clientConfig:
        service:
          name: kubernetes
          namespace: default
          path: /apis/{group}/{version}/{resource}
          port: 443
    ```

2. 在ApiService中，配置group、version，以及admission webhook的service：

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

1. API Server过滤指定的请求，并将其发给自己

2. 由aggregator转发到aggregated API server

这样就省去了为api server配置和维护kubeconfig文件的步骤。**但是服务端（admission webhook）的证书仍然需要自己生成和维护，并且设置API Service中的 `spec.caBundle` 字段，来指定 api server 使用的 CA 证书**。设置 `spec.insecureSkipTLSVerify` 为 true 则不使用TLS加密通信。

> 生产环境中，可以使用 [cert-manager](https://github.com/jetstack/cert-manager) 来自动生成和管理 TLS 证书，而不是直接存在 secret 资源对象中。
