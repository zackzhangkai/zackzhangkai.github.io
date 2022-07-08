---
layout: post
published: true
title: 使用kind在本地调试istio源代码
categories: [document]
tags: [istio go]
---
* content
{:toc}

## 前言

k8s sig开发了kind用于k8s开发，但是国内的环境让这个东西成为空谈。但是其实是有解决方法的，我们同样可以使用。本文告诉你如何使用gland + kind在mac上本地开发与调试istio。

## 背景

最近在本地调试istio，由于代码是master分支，而安装的集群是istio 1.6版本老版本；

istio从1.7开始，已经开始使用envoy v3版本的api，v2版本没有兼容，导致调试时经常失败。

因此想安装最新的istio来调试，但是原有的环境又不想破坏掉。因此需要有一个环境，安装最新版本的istio用于调试。

在亚太机器上使用kind构建了一套集群，1分钟时间。

如果要从本地访问这个集群，需要给该docker的apiserver证书中加入宿主机的外网IP。

但是由kind是使用docker起的一个K8s集群，没有找到如何来修改证书的方法。

>后来想到可以在该集群上通过iptables nat把流量源地址改成127.0.0.1，绕过apiserver证书对IP限制。但是并未去实践。

因此想在本地使用kind快速构建出集群，但是苦于国内糟糕的网络环境，如是有了本篇文章。

本篇文章，解决了国内使用kind普遍存在的拉取镜像慢、镜像拉取次受限导致无法痛快使用Kind集群的问题。

## 安装Kind

kind本身是个二进制命令，可以直接创建出一个k8s集群。

该集群是其实本质上是一个docker容器，因此启动速度很快(科学上网前提)，对宿主机环境依赖小。

一条命令搞定kind命令安装：

```bash
go get sigs.k8s.io/kind
```

## Mac Docker配置

由于依赖Mac本地Docker，在国内下载镜像速度慢，一般想到的方法是给docker配置代理。配置代理的方法：

```bash
# ~/.docker/config.json

{
 "proxies":
 {
   "default":
   {
     "httpProxy": "http://127.0.0.1:3001",
     "httpsProxy": "http://127.0.0.1:3001",
     "noProxy": "*.test.example.com,.example2.com,127.0.0.0/8"
   }
 }
}
```

加上代理后，拉取镜像到本地的速度确实快了不少。

但是受Dockerhub拉取次数限制。

因此最好的方法是配置私有仓库，然后让私有仓库从dockerhub拉取镜像。可以避免重复从dockerhub上拉取相同的镜像，且从私有仓库拉取走内网，速度很快。

在亚太环境上配置好私有registry仓库，然后本地配置registry到该地址。搭建私有创库的方法参考之前的文章。

本地docker 配置如下：

```bash
cat ~/.docker/daemon.json
{"debug":true,"experimental":false,"registry-mirrors": ["http://192.168.0.10:5000"]}
```

启动docker后，检查配置生效：

```bash
docker info

 Registry Mirrors:
  http://192.168.0.10:5000/
```

## 安装k8s集群

上面的配置就绪后，只剩安装K8s集群。

一条命令即可以创建一个集群：

```bash
kind create cluster
```

>注意：该命令会覆盖本地配置好的～/.kube/config

如果创建不成功，集群会自动清除；使用下面的命令，保留环境，输出日志方便debug

```bash
kind create cluster --retain
kind export logs
```

检查集群已经成功创建：

```bash
kubectl get po -A 

kube-system          coredns-f9fd979d6-6dxnx                      1/1     Running   0          3h2m
kube-system          coredns-f9fd979d6-v959w                      1/1     Running   0          3h2m
kube-system          etcd-kind-control-plane                      1/1     Running   0          3h2m
kube-system          kindnet-tvm2p                                1/1     Running   0          3h2m
kube-system          kube-apiserver-kind-control-plane            1/1     Running   0          3h2m
kube-system          kube-controller-manager-kind-control-plane   1/1     Running   1          3h2m
kube-system          kube-proxy-7xgr8                             1/1     Running   0          3h2m
kube-system          kube-scheduler-kind-control-plane            1/1     Running   0          3h2m
local-path-storage   local-path-provisioner-78776bfc44-45ww8      1/1     Running   0          3h2m
```

## 使用istio debug

首先打包istioctl

```bash
➜  istioctl git:(master) ✗ pwd
/Users/hugo/go/src/istio.io/istio/istioctl/cmd/istioctl
➜  istioctl git:(master) ✗ go build       
➜  istioctl git:(master) ✗ ls
doc.go           istioctl         istioctl_test.go main.go
```

build 出来istioctl后，安装：

```bash
➜  istioctl git:(master) ✗ ./istioctl install                                  
```

安装负载httpbin，注入sidecar用于测试：

```bash
➜ ✗ kubectl create deploy httpbin --image zackzhangkai/httpbin --port 80
➜ ✗ k label ns default istio-injection=enabled
```

template annotations中加入inject sidecar，重启注入sidecar

```bash
 ✗ k get po
NAME                      READY   STATUS    RESTARTS   AGE
httpbin-8d5d56c7d-nqgzn   2/2     Running   0          68m
```

启动pilot discovery

```bash
➜  pilot-discovery git:(master) pwd
/Users/hugo/go/src/istio.io/istio/pilot/cmd/pilot-discovery
➜  pilot-discovery git:(master) ls                                                                 
main.go    request.go var
➜  pilot-discovery git:(master) go run main.go discovery     
```

使用pilot cli检查xds

```bash
➜  debug git:(master) ✗ pwd
/Users/hugo/go/src/istio.io/istio/pilot/tools/debug

➜  debug git:(master) ✗ go run pilot_cli.go --type lds --proxytag httpbin-8d5d56c7d-nqgzn

 "typeUrl": "type.googleapis.com/envoy.config.listener.v3.Listener",
 "nonce": "Ixb+bcZpQIE=319bdf9c-adc4-4174-8d71-77cc801b14e2"

```

此时成功借助pilot cli完成了对pilot的简单调试。
