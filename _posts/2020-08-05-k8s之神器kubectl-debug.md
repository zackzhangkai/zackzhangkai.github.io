---
layout: post
published: false
title:  k8s之调试神器kubectl-debug
categories: [ k8s ]
tags: [troubleshooting]
---
* content
{:toc}

# kubectl 是什么？

`kubectl-debug`是调试K8s的命令行工具，具体它可以做什么呢？

首先想一下，你是不是有经常有这样的困惑：

+ 场景一：客户告诉你现在有个Pod日志中报错，连不上redis了。

然后你会怎么做？

是不是首先想，先进入这个pod，然后ping一下redis的IP?

好了，第一步：我们在输入繁琐的命令后，终于成功进入了这个pod。（当然，如果这个pod的安全性足够高，你可能连进入这个Pod的机会都没有，因为这个pod可能连sh这个命令都没有，这样你压根就进不去，直接GG了）

第二步：我们是不是要执行下ping命令？  好了，我们执行一下，发现连Ping都没有。这个时候是不是有想打人的冲动？（心中是不是会万马（草泥马）奔腾？）

此时我们的神器就要派上用场了。

kubectl debug解决的就是这种场景，它有个基础镜像，基础镜像中封装了我们常用的各种命令。然后结合你要debug的pod的镜像，相当于是在你的pod里面集成了所有的你要使用的镜像，然后你可能在里面可以执行各种你喜欢的操作。听起来是不是很爽？ 

确实很爽。但是如果对pod如何使用资源隔离的足够了解，这个命令只能说更方便了，但是我们也有第二选择，如nsenter，nsenter更强大的地方在于，它不仅可以直接模拟出pod的网络空间，还可以看到pod所有的文件，这个是不是更爽？


+ 场景二：如果客户告诉你，我环境不正常了，有个pod一直crashoff或是init无法成功。你要咋搞，是不是你的惯用伎俩，进入pod骚搞一把是不是不管用了？

好了，此时你也需要这个工具。


# 安装&使用

```bash
# curl -Lo kubectl-debug.tar.gz https://github.com/aylei/kubectl-debug/releases/download/v0.1.1/kubectl-debug_0.1.1_linux_amd64.tar.gz

# kubectl apply -f https://raw.githubusercontent.com/aylei/kubectl-debug/master/scripts/agent_daemonset.yml  # 这个是安装agent，daemonSet, 目的是快点，它会在集群的每个node上面起一个pod。

# such as ，加上 -a 表示agentless
kubectl debug -n kube-system kube-apiserver-ks-allinone -a

# 如果是init或是crash的pod，在后面加上--fork参数
```

好了，你可以去xjb搞了。