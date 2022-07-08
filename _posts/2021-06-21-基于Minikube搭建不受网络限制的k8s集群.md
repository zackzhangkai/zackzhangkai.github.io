---
layout: post
published: true
title: 基于Minikube搭建不受网络限制的k8s集群
categories: [k8s]
tags: [k8s,开发环境]
---
* content
{:toc}

## 为什么需要搭建minikube集群

在公有云公司工作最大的便利就是无需考虑外网限制，无镜像拉取的烦恼。

在青云KubeSphere团队工作进入倒计时，福利既然不能继续享受，搭建一个不受外网访问了限制的K8s集群势在必行。

Kind可以搭建一个更轻量的K8s集群，一分钟即能创建出一个新的集群，可以参考之前的文章。

Minikube 使用 docker 作为运行时，在虚机上创建一个all-in-one的k8s cluster，更接近一个真正的集群。

minikube优点：

1. 比kind健壮，更接近一个完整的K8s；
2. 使用docker，方便调试；
3. 可以很容易的暴露服务；
4. 如果要了解k8s的细节，可以直接登陆到minikube节点上排查。

## 搭建

`minikube` 是一个基于go语言二进制文件，由`k8s-sigs`开发，可以直接在 github 的 releases 上下载二进制。

Mac用户可以直接使用 `brew install minikube` 安装，本质也是把二进下载下来后，拷贝到PATH路径下。

启动：

```bash
# 配置http proxy 代理，配置完后即可拉取外网镜像
export http_proxy=http://1.2.3.4:1234
export https_http=http://1.2.3.4:1234

minikube start 
```

等待 minikube 起来后，ssh进虚机

```bash
minikube ssh
```

检查docker的配置

```bash
$ systemctl cat docker
# /usr/lib/systemd/system/docker.service
...
[Service]
...
Environment=HTTPS_PROXY=http://x.x.x.x:x
Environment=HTTP_PROXY=http://x.x.x.x:x
...

```

kubectl 是一个二进制文件，用来操作k8s集群，同理可以直接在github releases上下载二进制。

但其实minikube 直接集成了该命令，可以直接使用

```bash
alias k="minikube kubectl -- "
```

配置自动补全

```
source <(minikube completion zsh)
```

## 配置私有镜像仓库

由于dockerhub的镜像拉取策略限制，很容易造成镜像拉取失败，可以配置一个私有镜像仓库作为缓存；

一般在 ～/.docker/daemon.json配置可生效

```bash
# cat ~/.docker/daemon.json 
{"debug":true,"experimental":false,"registry-mirrors": ["http://x.x.x.x:5000"]}
```

但是对于minikube并不生效，可以直接在dockerd命令后面接上相应参数即可

```bash
# systemctl cat docker| head
# vi /usr/lib/systemd/system/docker.service
...
ExecStart=/usr/bin/dockerd --registry-mirror http://x.x.x.x:5000 ....  
```

## 导入本地私有镜像

```bash
minikube cache add zackzhangkai/minikube-nginx-ingress-controller:v0.40.2
minikube cache add jettech/kube-webhook-certgen:v1.3.0
minikube cache add jettech/kube-webhook-certgen:v1.2.2
```

至此，本地开发环境已经搭建完毕，可以拉取gcr.io aws dockerhub的镜像。
