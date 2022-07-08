---
layout: post
published: true
title:  Promethus与zabbix的优劣对比
categories: [document]
tags: [promethus]
---
* content
{:toc}

# promethus
## 简介
Prometheus 是一套开源的系统监控报警框架。它启发于 Google 的 borgmon 监控系统，由工作在 SoundCloud 的 google 前员工在 2012 年创建，作为社区开源项目进行开发，并于 2015 年正式发布。2016 年，Prometheus 正式加入 Cloud Native Computing Foundation，成为受欢迎度仅次于 Kubernetes 的项目。

## 特点
1. 强大的多维度数据模型
  - 时间序列数据通过 metric 名和键值对来区分。
  - 所有的 metrics 都可以设置任意的多维标签。
  - 数据模型更随意，不需要刻意设置为以点分隔的字符串。
  - 可以对数据模型进行聚合，切割和切片操作。
  - 支持双精度浮点类型，标签可以设为全 unicode。

2. 灵活而强大的查询语句（PromQL）：在同一个查询语句，可以对多个 metrics 进行乘法、加法、连接、取分数位等操作。

3. 易于管理： Prometheus server 是一个单独的二进制文件，可直接在本地工作，不依赖于分布式存储。
4. 高效：平均每个采样点仅占 3.5 bytes，且一个 Prometheus server 可以处理数百万的 metrics。
5. 使用 pull 模式采集时间序列数据，这样不仅有利于本机测试而且可以避免有问题的服务器推送坏的 metrics。
6. 可以采用 push gateway 的方式把时间序列数据推送至 Prometheus server 端。
7. 可以通过服务发现或者静态配置去获取监控的 targets。
8. 有多种可视化图形界面。
9. 易于伸缩。

> **注意：由于数据采集可能会有丢失，所以 Prometheus 不适用对采集数据要 100% 准确的情形。但如果用于记录时间序列数据，Prometheus 具有很大的查询优势，此外，Prometheus 适用于微服务的体系架构**

## 组件
promethus的组件是可选的

+ Prometheus Server: 用于收集和存储时间序列数据。
+ Client Library: 客户端库，为需要监控的服务生成相应的 metrics 并暴露给 Prometheus server。当 Prometheus server 来 pull 时，直接返回实时状态的 metrics。
+ Push Gateway: 主要用于短期的 jobs。由于这类 jobs 存在时间较短，可能在 Prometheus 来 pull 之+ 前就消失了。为此，这次 jobs 可以直接向 Prometheus server 端推送它们的 metrics。这种方式主要用+ 于服务层面的 metrics，对于机器层面的 metrices，需要使用 node exporter。
+ Exporters: 用于暴露已有的第三方服务的 metrics 给 Prometheus。
+ Alertmanager: 从 Prometheus server 端接收到 alerts 后，会进行去除重复数据，分组，并路由到对收的接受方式，发出报警。常见的接收方式有：电子邮件，pagerduty，OpsGenie, webhook 等。

![](/styles/images/promethus_架构图.png)


## 工作流程
1. Prometheus server 定期从配置好的 jobs 或者 exporters 中拉 metrics，或者接收来自 Pushgateway 发过来的 metrics，或者从其他的 Prometheus server 中拉 metrics。
2. Prometheus server 在本地存储收集到的 metrics，并运行已定义好的 alert.rules，记录新的时间序列或者向 Alertmanager 推送警报。
3. Alertmanager 根据配置文件，对接收到的警报进行处理，发出告警。
4. 在图形界面中，可视化采集数据。

# zabbix
Zabbix是一个高度集成的网络监控解决方案，一个简单的安装包中提供多样性的功能。

## 功能特点
1. 数据收集
  - 可用性和性能检查
  - 支持SNMP（包括主动轮训和被动获取），IPMI，JMX，VMware监控
  - 自定义检查
  - 按照自定义的间隔收集需要的数据
  - 通过server/proxy+agents来执行

2. 网络发现
  - 自动发现网络设备
  - 监控代理自动注册
  - 发现文件系统，网络接口和SNMP OID值
