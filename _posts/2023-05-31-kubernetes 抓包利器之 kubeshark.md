---
layout: post
published: true
title: 2023-05-31-kubernetes 抓包利器之 kubeshark
categories: [document]
tags: [云原生,kubeshark,kubernetes,抓包]
---
* content
{:toc}

# kubeshark 介绍

先上一个截图看看效果：


![官方 shark 图](/images/kubeshark-ui.png) 

通过这个图，可以看到很清晰的看到集群中所有流量访问的抓包。

下面的图是装上 istio 的 bookinfo 做测试的结果：

![过滤 productpage 访问路径的http 包](/images/kubeshark-productpage.jpg) 

不仅可以看到请求的 header，还可以看到访问到的 k8s 的资源的 yaml，如下：

![工作负载manifest](/images/kubeshark-productpage-manifest.jpg) 

查看访问到该节点的流量图：
![流量图](/images/kubeshark-servicemap.jpg) 


到此，已经基本展示了抓包内容，以及该包访问到 k8s 上的负载相关信息（如 负载 ip，pod所在节点 等）。

总结 kubeshark 所具有的功能：
  1. 包含丰富的过滤规则，类似于 tcpdump ；
  2. 能直观的展示http请求头、body、response，类似于 wireshark，但是比 wireshark 直观、方便；
  3. 包含流量向图，可以直观看到访问流量大小。

看到这，我们已经被它强大的功能吸引了，感叹技术提升效率能节约多少生命！

# 开始

参考[官网](https://docs.kubeshark.co/en/introduction)：

从 [github release](https://github.com/kubeshark/kubeshark/releases) 界面下载二进制安装：

1. mac安装方式：

```bash
curl -Lo kubeshark https://github.com/kubeshark/kubeshark/releases/download/40.5/kubeshark_darwin_amd64 && chmod 755 kubeshark
```

2. windows 安装方式：

```bash
curl -LO https://github.com/kubeshark/kubeshark/releases/download/40.5/kubeshark.exe
```

3. linux 安装方式：

```
curl -Lo kubeshark https://github.com/kubeshark/kubeshark/releases/download/40.5/kubeshark_linux_amd64 && chmod 755 kubeshark
```

安装完后，一条命令就可以启动kubeshark deployment:

```
ks='kubeshark tap --docker-registry image.zackzhangkai.cn/csm/kubeshark/'
```

>注意：如上述，加上内网镜像仓库，可以在离线集群上抓集群流量包

安装完后，如下：

```
$kubectl -n kubeshark get all
NAME                                    READY   STATUS    RESTARTS   AGE
pod/kubeshark-front                     1/1     Running   0          160m
pod/kubeshark-hub                       1/1     Running   0          160m
pod/kubeshark-worker-daemon-set-bpmhv   1/1     Running   1          160m
pod/kubeshark-worker-daemon-set-nscbq   1/1     Running   5          160m
pod/kubeshark-worker-daemon-set-z5nbg   1/1     Running   2          160m

NAME                      TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/kubeshark-front   NodePort   10.233.21.183   <none>        80:32262/TCP   160m
service/kubeshark-hub     NodePort   10.233.11.141   <none>        80:30090/TCP   160m

NAME                                         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/kubeshark-worker-daemon-set   3         3         3       3            3           <none>          160m
➜  kubeshark_40.5_darwin_amd64
```

待上述 Pod 全部 running 后，会自动打开 proxy 页面，显示所有 pod 中的流量。

只要这些 pod 一直在运行，下次进入页面，可以直接执行 `kubeshark proxy` 即可重抓包并过滤包规则。