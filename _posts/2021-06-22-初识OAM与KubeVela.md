---
layout: post
published: true
title: 初识OAM与KubeVela
categories: [k8s]
tags: [k8s,kubevela,oam]
---
* content
{:toc}

## 为什么要研究oam/kubevela?

Kubernetes在与mesos、swarm的竞争中获胜，已经是事实上的云原生操作系统。

***但是这个平台太复杂了***

**业务开发部门：你们喜欢折腾，跟我有啥关系？**  

- 对于业务开发来说，他们聚焦在业务逻辑、订单流转等功能实现上，如电商平台，订单系统、仓库系统、支付系统等。  
- 他们不会太在意底层使用什么系统，你甚至用windows把他的tomcat服务跑起来就行。  
- 因为稳定性是架构部门、或者系统部门或者是运维的事情。  

**基础平台部：该如何说服业务开发部门来使用k8s系统呢？**  

- 你跟他们讲一大堆k8s的好处，什么服务发现、什么负载均衡、什么自动伸缩，balabala...  
- 他们其实并不关心！他们说spring cloud有全家桶，什么都有，而且社区成熟繁荣。

**现状：大多业务部门，迫于公司领导对云原生要求，要求业务开发迁移至k8s平台。**  

- ***k8s是一个极其复杂的系统，cluster、pod、deployment、statefulset、hpa、ingress、istio、service、镜像、docker等概念已经足以让人眼花缭乱，而且还涉及网络、存储等底层的技术。***  
- 如果要求业务开发人员掌握，相信他们内心是崩溃的。  
- 迁移成本极其巨大！（时间成本、沟通成本、学习成本）  

**有没有一种方案，能降低业务研发的心智负担，能快速让他上手？**

- 业界有成功的典型：`Heroku`。  
- 开发人员只需要把自己的代码地址指定下，就可以把它的服务跑出来。  
- 国内也有类似的产品，如：`Rainbond`。  


这就是OAM的来源。  

![Optimized-Kubernetes-Expectation-vs-Reality-1](/styles/images/Optimized-Kubernetes-Expectation-vs-Reality-1.png)

## 什么是OAM?

OAM（Open Application Model）是阿里巴巴和微软共同开源的云原生应用规范模型，同时开源了基于 OAM 的实现 Rudr， 2019 年 10 月宣布开源。

提供开发人员与运维人员两个视角，开发人员不用关注复杂而庞大的Kubernetes，只用关注业务开发。如开发完应用后，要想让这个应用跑起来，只需要定义如：4c8g 这些最基本的参数。不用去定义k8s中的request、镜像、特权等。每个组件称之为componet。一个应用可以有多个component。

运维人员根据开发人员的提供的参数，把应用负责部署出来。给componet加上应用监控、灰度发布、熔断限流等操作，称之为Traint。每个Component可以有多个Traint。

有些compoent开发需要告诉运维指标，如要求响应100ms以内，开发直接把自己的要求写在这些policy。开发不用去管运维用哪些Traint来保证这些SLA。一切数据化、可度量。

Rudr把Trait解析成K8s的yaml，采用rust开发。

目前有两个版本：1.0.1版本已经归档不维护。

**基于OAM 1.0.0-alpha1的rudr实现**

- rudr本身是由rust语言开发。  
- 将OAM概念映射成CRD，而rudr controller则负责调谐这个映射关系。  
- 内置traits: Ingress, Autoscaler, Manual Scaler，其他traits可以尝试从helm hub获取。  

**基于OAM 1.0.1-alpha2的crossplane实现**

- crossplane本身是由go语言开发。  
- 安装crossplane成为在k8s集群使用OAM的前提。  
- OAM 1.0.1-alpha2与crossplane强依赖。  

## 什么是Kubevela?

oam是API的定义，是一种标准，是资源模型，即resource model；

kubevela是核心组件，是controller。

[Github地址](https://github.com/oam-dev/kubevela)

[官网](http://kubevela.io/)

## 什么是Component?

一个app由多个component组成，component需要自己的属性（镜像、端口）、trait、type等；component有许多类型，如：前端（webservice）、后端（worker）、定时任务（task）；这些类别就是comp（componentdefinitions）；component是一个实际执行的模块单元，它还有版本控制，即comprev；

`comp`与`component`是不同的两个资源，`comp`是`component`的类别。

```bash
✗ kubectl get comp -n vela-system
NAME         WORKLOAD-KIND   DESCRIPTION
task         Job             Describes jobs that run code or a script to completion.
webservice   Deployment      Describes long-running, scalable, containerized services that have a stable network endpoint to receive external network traffic from customers.
worker       Deployment      Describes long-running, scalable, containerized services that running at backend. They do NOT have network endpoint to receive external network traffic.
```

## 什么是trait?

trait是这个component的各种能力，如sidecar、ingress、hpa等。

**注意sidecar不会修改deployment的template中container定义的yaml，而是直接通过trait就能生效。**

```bash
✗ kubectl get trait -n vela-system
NAME          APPLIES-TO                DESCRIPTION
annotations   ["webservice","worker"]   Add annotations for your Workload.
cpuscaler     ["webservice","worker"]   Automatically scale the component based on CPU usage.
ingress       ["webservice","worker"]   Enable public web traffic for the component.
labels        ["webservice","worker"]   Add labels for your Workload.
scaler        ["webservice","worker"]   Manually scale the component.
sidecar       ["webservice","worker"]   Inject a sidecar container to the component.
```
