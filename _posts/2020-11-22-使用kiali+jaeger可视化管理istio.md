---
layout: post
published: true
title:  使用kiali+jaeger可视化管理istio
categories: [document]
tags: [k8s,docker,kiali,istio,微服务]
---
* content
{:toc}

## 概念

### 什么是kiali?

Kiali是uber开源的软件，基于prometheus对istio实现流量的可视化；支持直接对istio crd（Detination/Virtualservice）的操作，实现流量规则的在线编辑，实现流量的控制。

### 什么是Jaeger

Jaeger是流量分布式Tracing，主要解决的场景是将传统的单体服务被拆分成不同的微服务后，流量访问节点增多后可视化追踪的问题。传统的单体服务，对外暴露一个端口，提供一个访问路径，客户端只需要请求这个一接口就可以获取数据。但是当其中一个较小的故障时，就会导致整个服务不可用，难以满足互联网高并发的场景。如果将这个庞大的服务进行拆分成不同的模块的微小型服务后，又会使整个访问调用链变得复杂。一个请求进来后，经过N个递归调用，最终获得数据。在这么多的调用时，如果有一次的调用出了问题，如何快速定位问题，就显得非常有必要。同时，如果知道每一次的调用时，请求响应时间，就能帮助开发人员找出瓶颈所在，并优化程序。

在使用Jager时，在请求进来后，会生成一个TracingID，放入Http Header头部，该TracingID会伴随请求整个生命周期，利用该ID，绘制出所有的请求的流量拓扑，包含每一跳的响应时间及正常到达情况。

## Kiali安装

提供两种安装方式

### 使用 kiali operator方式安装

1. 使用operator来安装

```bash
helm repo add kiali https://kiali.org/helm-charts
helm search repo kiali
helm pull kiali/kiali-operator
helm install kiali-operator kiali-operator-1.26.1.tgz -n istio-system
```

2. 部署了Operator之后，需要部署一个[kiali cr资源](https://github.com/kiali/kiali-operator/blob/master/deploy/kiali/kiali_cr.yaml
)，cr中定义kiali的配置及版本，就部署出kiali的pod。

Operator监控该cr中的值，当发生变化时，会动态来调整kiali pod来达到与定义的配置一致性，即调谐的过程。

Operator本质上是使用ansible来操作资源。

我的cr文件如下，其中自定义了prometheus的路径，默认是istio-system下的prometheus：

```bash
apiVersion: kiali.io/v1alpha1
kind: Kiali
metadata:
  namespace: istio-system
  name: kiali
  annotations:
    ansible.operator-sdk/verbosity: "1"
spec:
  istio_component_namespaces:
    prometheus: kubesphere-monitoring-system

  # istio_namespace: "istio-system"
  view_only_mode: true
  auth: anonymous
  prometheus:
    cache_duration: 10
    cache_enabled: true
    cache_expiration: 300
    url: "http://prometheus-k8s.kubesphere-monitoring-system.svc:9090"
```

由于kiali的operator有问题，部署出来后，代码一直报错，如[issue](https://github.com/kiali/kiali/issues/3459)。

报错原因是：kiali这个命令在启动的时候，打印日志的参数`-v`不支持，尝试更换几个版本后没有解决，不知道Kiali官方是怎么测试的。因此我的做法是直接把deployment的yaml获取到后，把这个参数去掉，然后直接`apply deployment yaml`文件就正常了。

### 直接使用chart安装kiali server

```bash
 $ helm search repo kiali
NAME                	CHART VERSION	APP VERSION	DESCRIPTION
kiali/kiali-server  	1.26.1       	v1.26.1    	Kiali is an open source project for service mes...

$ helm install kiali kiali-server -f custom-values.yaml
```

由于我的prometheus及Kiali的配置并非默认，因此需要自定义，values.yaml配置文件如下：

```bash
# custom-values.yaml

nameOverride: "kiali"
fullnameOverride: "kiali"

istio_namespace: "istio-system" # default is where Kiali is installed

auth:
  strategy: "anonymous"

deployment:
  accessible_namespaces:
  - "**"
  additional_service_yaml: {}
  affinity:
    node: {}
    pod: {}
    pod_anti: {}
  custom_dashboards:
    excludes: ['']
    includes: ['*']
  image_name: quay.io/kiali/kiali
  image_pull_policy: "Always"
  image_pull_secrets: []
  image_version: v1.26.1
  ingress_enabled: true
  logger:
    log_format: "text"
    log_level: "info"
    time_field_format: "2006-01-02T15:04:05Z07:00"
    sampler_rate: "1"
  node_selector: {}
  override_ingress_yaml:
    metadata: {}
  pod_annotations: {}
  pod_labels: {}
  priority_class_name: ""
  replicas: 1
  resources: {}
  secret_name: "kiali"
  service_annotations: {}
  service_type: ""
  tolerations: []
  version_label: v1.26.1
  view_only_mode: false

external_services:
  prometheus:
    url: http://prometheus-k8s.kubesphere-monitoring-system:9090
  tracing:
    in_cluster_url: http://jaeger-query.istio-system.svc:16686
  custom_dashboards:
    enabled: true

server:
  port: 20001
  metrics_enabled: true
  metrics_port: 9090
  web_root: ""
```

1. ui访问 kiali，可以直接把pod的端口暴露下，可以使用port-forwad，也可以使用svc NodePort方式。

效果如下：

查询所有的namespace下的流量拓扑图

![picture 2](/images/f42cdad59de160d26da11cd57b0be070d8f63ae0b9edcc43381024c3034c4f39.png)  

功能比较强大，还可以直接修改istio DestinationRules

![picture 3](/images/7fbd99e37b22fa633fbed95781d6404a21597604b82bb17db058fd88ab9a7c9c.png)  

修改VirtualService

![picture 5](/images/1774e06048a74178ce5610fd9ee7b1f3dbfb298e2f4072d7f5093f192a67b4e9.png)  

与Jaeger的集成

![picture 3](/images/a761734dce84dfdc7ce8e4e5a3b66360fc8ff0cee9984205f6b637ac87c19b08.png)  

![picture 4](/images/8f144a40aea011e52a260584c2ec51c7e54aa058da575987a6b0a784a5b5dc96.png)  
