---
layout: post
title:  kubernetes proxy
date:   2018-04-26 01:08:00 +0800
categories: document
tag:
  - kubernetes

---
* content
{:toc}

# 前言
在研究service时，想知道，如果想把pod直接expose后，此时服务类型为ClusterIp，不知道是否能通过kubectl proxy能让外部访问该服务？带着这个疑问，来研究一下kubectl proxy的具体原理


# kubectl proxy
kubectl proxy其实就是暴露API的
```
kubectl run node-hello --image=gcr.io/google-samples/node-hello:1.0 --port=8080
kubectl proxy --port=8080
curl localhost:8001
```
结果：
```
...
"/apis/",
"/apis/admissionregistration.k8s.io",
"/apis/admissionregistration.k8s.io/v1beta1",
"/apis/apiextensions.k8s.io",
"/apis/apiextensions.k8s.io/v1beta1",
...
```
