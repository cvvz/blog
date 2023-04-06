---
title: "Prometheus试玩"
date: 2023-04-05T09:26:59+08:00
draft: false
comments: true
toc: true
autoCollapseToc: false
keywords: []
tags: []
---

## Mental Model
1. Prometheus的[配置文件](https://prometheus.io/docs/prometheus/latest/configuration/configuration/)中，[scrape_config](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#scrape_config) 部分定义的是[job](https://prometheus.io/docs/concepts/jobs_instances/#jobs-and-instances) —— 也就是去哪个target [instance](https://prometheus.io/docs/concepts/jobs_instances/#jobs-and-instances) scrape metrics —— 但是**配置文件里不关心也没法关心具体抓哪些指标**。

2. 具体有哪些指标可以抓，也就是 **[metrics name + label](https://prometheus.io/docs/concepts/data_model/#metric-names-and-labels) 和 [metrics type](https://prometheus.io/docs/concepts/metric_types/)，都在客户端源代码中定义**。

> 如果configuration没有配置对，相当于找不到抓的地方。
> 
> 如果client代码不正确，相当于抓的地方对了，但是什么东西都抓不到。

3. 抓到metrics以后，就可以定义rules —— 通过[promQL查询](https://prometheus.io/docs/prometheus/latest/querying/basics/)，然后处理查询结果 —— [alerting](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/)或是[recording](https://prometheus.io/docs/prometheus/latest/configuration/recording_rules/)

> 报警无法触发的原因有两种：
> 
> * client或configuration没写对
> * promQL没写对，可以使用ut测试rules规则是否正确。

## configuration
除了直接配置target instance之外，scrape_config还支持多种服务发现机制，[kubernetes_sd_config](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#kubernetes_sd_config)是k8s的服务发现配置

服务发现机制会给metrics额外设置一些`__meta__`开头的label，可以通过[relabel_config](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#relabel_config)来过滤出具体的instance，比如：

```yaml
relabel_configs:
  - target_label: __address__
    replacement: $(FQDN):443
  - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_pod_label_component,     __meta_kubernetes_pod_container_name]
    action: keep
    regex: kube-system;kube-proxy;kube-proxy
  - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_pod_name]
    regex: (.+);(.+)
    target_label: __metrics_path__
    replacement: /api/v1/namespaces/${1}/pods/${2}:10249/proxy/metrics
  - source_labels: [__meta_kubernetes_pod_name]
    regex: (.*)
    target_label: pod
    action: replace
  - source_labels: [__meta_kubernetes_namespace]
    regex: (.*)
    target_label: namespace
    action: replace
```

prometheus会抓取 `kube-system/kube-proxy`的metrics，target instance是`$(FQDN):443`，metrics api path是`/api/v1/namespaces/kube-system/pods/kube-proxy:10249/proxy/metrics`

> 说明kube-proxy的metrics并不是直接从pod的/metrics接口获取的，而是通过[apiserver的proxy api](https://kubernetes.io/docs/tasks/access-application-cluster/access-cluster-services/#manually-constructing-apiserver-proxy-urls))

并且给metrics加上`namespace`和`pod`这两个label，比如：

{{< figure src="/prometheus/kubeproxy.png" width="1000px" >}}




## promQL

promQL中有两个非常常见，但又相对复杂的query，一个是计算占比（ratio），一个是计算百分位（quantile）

### 计算ratio

```yaml
# ratio of api server requests over 5m
- record: job_verb_code_instance:apiserver_request:ratio_rate5m
    expr: |
    sum by(job, verb, instance) (rate(apiserver_request_total[5m]))
        / ignoring (verb) group_left()
    sum by(job, instance) (rate(apiserver_request_total[5m]))
```

首先通过`apiserver_request_total[5m]`得到5m内的全部采样点，即`range vector`，然后用rate计算每秒增长速率，最后用sum聚合，两边做除法就得到比例关系。

如果只执行左边：
{{< figure src="/prometheus/1.png" width="1000px" >}}

如果只执行右边：
{{< figure src="/prometheus/2.png" width="1000px" >}}

可以看到因为左边的query带上了更多的label，因此可以查到更多的数据。

我们想计算每一种verb在所有请求中的占比，先使用 [ignoring](https://prometheus.io/docs/prometheus/latest/querying/operators/#vector-matching-keywords) 忽略掉`verb`label，这样两边label一致才能做除法；

因为左边的数据多(many)，右边的数据只有一条(one)，这属于[Many-to-one matches](https://prometheus.io/docs/prometheus/latest/querying/operators/#many-to-one-and-one-to-many-vector-matches)，所以接着用`group_left()`让左边的每一条数据依次除以右边，得到最终结果：

{{< figure src="/prometheus/3.png" width="1000px" >}}


### 计算quantile

```yaml
# 99th percentile latency
- record: job:apiserver_request_latency:99pctlrate5m
  expr: |
    histogram_quantile(0.99, sum by (le, job)(rate(apiserver_request_duration_seconds_bucket{verb=~"GET|POST|DELETE|PATCH"}[5m]))) * 1000 > 0
```

首先`apiserver_request_duration_seconds_bucket`是一个bucket，通过`[5m]`得到每一个bucket在5m内的采样值，然后计算每一个bucket中的增长rate，再按照le(less equal，即桶的上边界)进行sum聚合。最后通过[histogram_quantile](https://prometheus.io/docs/prometheus/latest/querying/functions/#histogram_quantile)函数计算百分位。

>**⚠️注意，histogram不会记录真实数据，只记录每个 bucket 下的 count 和 sum，然后假定数据在桶中是均匀分布的来计算百分位。如果 bucket 设置的不合理，会产生不符合预期的 quantile 结果:**
> 
> 比如bucket 是 100ms ~ 1000ms，而大部分记录都在 100ms ~ 200ms 之间，但是由于prometheus假定数据是均匀分布的，因此计算 P99 时会得到接近于 1000ms 的值。[官方文档](https://prometheus.io/docs/practices/histograms/#errors-of-quantile-estimation)里描述了这个问题。



## 几种时间的理解
Prometheus里还涉及到几种时间概念，必须搞清楚才能知道怎么样设置rule才能正确触发告警

### evaluation_interval, scrape_interval和alerting rules中的for
- **evaluation_interval**：只有当它 >= scrape_interval时，才能确保每次evaluation时，promQL查询到的都是新数据。
- **alerting rule中的for**：和上面同理，只有当它 >= scrape_interval时，才能确保告警发出时至少覆盖了两次sample。

> 1. 当for < evaluation_interval 时，必须要Pending到第二次evaluation发生时才能判断是否真的要fire alert
> 
> 2. evaluation_interval和for，其中任何一个 >= scrape_interval，就可以保证告警发出时至少覆盖了两次sample，不会有false alert。

### range vector中的[time duration]
以`[10m]`为例，意思是从查询时刻算起，往前推10m，时序数据库里存了多少数据就查多少数据出来，意味着：

- **if scrape_interval == 10m**，那么基本上可以保证你刚好只能查出来一条数据；
- **if scrape_interval > 10m**，那么你有可能查不出来任何数据；
- **if scrape_interval < 10m**，你可能可以查出来多条数据。采样间隔小于5m，你肯定可以查出来2条数据，依次类推

> 在写prometheus的ut时，如果promQL计算的是range vector的数据类型，最好避免evaluation_interval和input_series的interval（即scrape_interval）成**倍数关系**。
> 
> 这是因为在ut中，evaluation_interval和scrape_interval的开始时间都是0，如果这两个interval是倍数关系，那么在evaluation的时候，总是会有时刻，evaluation时刚好产生一个input，计算时会把这个input也带进去，这可能会产生让人困惑的ut结果。这种情况在真实场景中很难发生。