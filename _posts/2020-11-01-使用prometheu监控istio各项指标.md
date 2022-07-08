---
layout: post
published: false
title:  使用prometheus监控istio各项指标
categories: [document]
tags: [prometheus,istio,微服务]
---
* content
{:toc}

## Istio安装

## Prometheus安装

## Prometheus配置文件

[Prometheus配置文件](https://github.com/zackzhangkai/ks-installer/blob/ec301d2ede8592d38e1cbb606f1b8634f7fab39f/roles/ks-istio/files/prometheus/prometheus-additional.yaml)

```bash
# Scrape config for envoy stats
- job_name: 'envoy-stats' # Prometheus的监控，按job来区分，每一个job定义了要抓取metric的配置
  scrape_interval: 15s  # 采样周期
  metrics_path: /stats/prometheus # Path，如curl nodeIP:15090/stats/prometheus
  kubernetes_sd_configs:  # K8s 服务发现配置，目前支持5种：pod/service/endpoint/node/ingress
  - role: pod

  relabel_configs:
  - source_labels: [__meta_kubernetes_pod_container_port_name]
    action: keep
    regex: '.*-envoy-prom'
  - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port] # 这个对应的是pod的annotation如：prometheus.io/port: "10254"
    action: replace
    regex: ([^:]+)(?::\d+)?;(\d+) # $1对应IP, $2对应Port
    replacement: $1:15090 # 15090是sidecar的prometheus的监听端口，上面的10254是nginx ingress的监听端口；这里做了replace，转成sidecar的prometheus端口。
    target_label: __address__ # 在Prometheus页面的服务发现可以看到这个配置 __address__="10.233.75.38:15090"
  - action: labeldrop
    regex: __meta_kubernetes_pod_label_(.+)
  - source_labels: [__meta_kubernetes_namespace]
    action: replace
    target_label: namespace
  - source_labels: [__meta_kubernetes_pod_name]
    action: replace
    target_label: pod_name
```

![配置信息](/images/2020-12-20-12-56-37.png)

![graph](/images/2020-12-20-12-58-49.png)

![告警信息](/images/2020-12-20-12-59-33.png)

[参考1](https://yunlzheng.gitbook.io/prometheus-book/part-iii-prometheus-shi-zhan/readmd/service-discovery-with-kubernetes)

[参考2](https://prometheus.io/docs/prometheus/latest/configuration/configuration/)
