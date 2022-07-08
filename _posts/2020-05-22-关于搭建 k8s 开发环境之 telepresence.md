---
layout: post
published: true
title:  k8s开发环境之 telepresence
categories: [ k8s ]
tags: [ telepresence ]
---
* content
{:toc}

# 什么是 telepresence?

k8s 的开发环境，比如：当开发一个新的 k8s 的 feature 或是修复一个 bug 时，修改完 golang 代码后，需要快速验证下这个代码的功能是否达到了预期，如果不借助工具，那么一般思维是：

change code -> rebuild image -> push image to docker registry - redeploy k8s deployment

但是，这太慢了。

此时 telepresence 就派上了用场。它的功能是可以在本地直接启动新的服务后，直接替换 k8s(本地或是远程)上的某个特定的deployment。

# 快速搭建k8s cluster 及环境及配置

k8s 的环境是必不可少的。

本地可以用 minikube

```
brew install minikube
brew install kubectl
```
然后就会在本地生成 `～/.kube/config`文件，然后本地就可以用 `kubectl` 来操作环境。

>建议备份下这个 config 文件，如果发生修改了，可以直接还原回来。如果没有备份，又被修改了，导致 kubectl 执行命令失败。可以通过关机 minikube，然后开机就可以恢复。
>`minikube stop && minikube start`

远程的 k8s 可以用 kubesphere 的 all-in-one来快速搭建。

搭建完之后，本地要想通过 kubectl 命令来操作远程的 K8s cluster，如何操作呢？

## 利用本地 kubectl 来操作远程 k8s cluster

直接把远程 k8s cluster 的~/.kube/config 拷备到本地并替换。

但是还是无法通过命令访问，为什么呢？

原因是这样的：比如你在青云的云平台上创建了个主机，然后该主机有自己的内网 IP  192.168.0.x 。然后你还要申请并绑定个 eip 才能访问外网。但是这个 eip 并不是该主机的的 eth 上的 ip。也就是在这个主机执行 `ip ad`上你是看不到这个 eip 的。这个时候我部署的 k8s cluster它用的 ip 是 192.168.0.x这个内网 IP。同时，在生成的 ~/.kube/config 里面也是这个 IP。因此即使拷备到了本地，网络也是不通的，还是不通的，如果手动把这个 IP改成 eip，这个时候网络是通了，但是会提示 cert/key 证书不对。

要怎么办呢？

这个时候其实只需要搭建个代理就行了。

在这个远程主机上通过 `ncat -l --proxy-type http localhost 81`。执行完这个，就搭建了个 http 的正向代理。

然后在 iterm2 本地，执行

`export http_proxy="http://eip:81"; export HTTP_PROXY="http://eip:81"; export https_proxy="http://eip:81"; export HTTPS_PROXY="http://eip:81"`

这样就把本地的 http(s)的流量代理到了这个主机上，然后此时就可以像在远程环境上一样，正常使用 kubectl 这个命令了。


# telepresence的安装与使用

如果要用 teletepresence debug 远程k8s，一般要按上述方法来配置从本地通用 kubectl 来操作远程 k8s 环境。(当然 teletepresence也支持加上代理的参数。)

## 安装

```
brew install teletepresence
```
也可以直找到 github 的 release，直接下载一个包，解压后是二进制文件`etcd kube-apiserver kubebuilder kubectl`。把这些二制拷备到系统环境变量下即可。

## debug 本地 k8s cluster

首先把 ～/.kube/config 下的文件替换成本地的配置。

确认能够正常用 kubectl 命令了。

```
$ kubectl create deployment hello-world --image=datawire/hello-world
$ kubectl expose deployment hello-world --type=LoadBalancer --port=8000
```
然后看下 service
```
hugo@zack:/Users/hugo/.kube $ kubectl get svc hello-world
NAME          TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
hello-world   LoadBalancer   10.101.139.40   <pending>     8000:32293/TCP   63m
```
可以看到这个externalIP 是 pending 的，官网让你一直等下去，其实不对。要 `edit servce`把它改成 `NodePort`或是加上`externalIPs`。

```
hugo@zack:/Users/hugo $ kubectl edit svc hello-world
service/hello-world edited
hugo@zack:/Users/hugo $ kubectl get svc -w
NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
hello-world   NodePort    10.101.139.40   <none>        8000:32293/TCP   141m
```
看下本地的 node 的 ip
```
hugo@zack:/Users/hugo $ minikube node list
minikube	192.168.64.2
```
从本地可以 curl 到内容
```
hugo@zack:/Users/hugo $ curl 192.168.64.2:32293
Hello, world!
```
在本地启动 http web server,并能成功 curl 成功
```
$ python3 -m http.server 8002 &
$ curl localhost:8002
$ kill %1
```

此时，把这个这个本地 web 用 telepresence替换到k8s 环境上的 hello-world 服务。
```
$ telepresence --swap-deployment hello-world --expose 9090   --run python3 -m http.server 9090
```
这个命令会把hello-world 这个deploy 给 scale 到 0，然后替换成 teletepresence的
```
hugo@zack:/Users/hugo $ kubectl get deploy -w
NAME                                           READY   UP-TO-DATE   AVAILABLE   AGE
hello-world                                    0/0     0            0           157m
hello-world-1ea1add6ad3f4c91a4a2a24435ad66cb   0/1     1            0           108s
```
发现 deploy 替换后一直不正常，describe pod 发现是拉镜像`datawire/telepresence-k8s:0.105`不成功。

关闭 minikube然后重新启动，看下能不能找到一些提示。

果然有些提示：

```
 To pull new external images, you may need to configure a proxy: https://minikube.sigs.k8s.io/docs/reference/networking/proxy/
```

提示要配置代理
```
 $ export http_proxy="http://ss:81"; export HTTP_PROXY="http://ss:81"; export https_proxy="http://ss:81"; export HTTPS_PROXY="http://ss:81";NO_PROXY=localhost,127.0.0.1
 $ minikube start
```
这个时候就正常了。

```
hugo@zack:/Users/hugo $ kubectl get deploy -w
NAME                                           READY   UP-TO-DATE   AVAILABLE   AGE
hello-world                                    0/0     0            0           3h4m
hello-world-8fc02edba0a34faf947bf77f659da50d   1/1     1            1           22s
```

```
hugo@zack:/tmp/telepresence-test $ telepresence --swap-deployment hello-world --expose 9090   --run python3 -m http.server 9090
T: Starting proxy with method 'vpn-tcp', which has the following
T: limitations: All processes are affected, only one telepresence can
T: run per machine, and you can't use other VPNs. You may need to add
T: cloud hosts and headless services with --also-proxy. For a full
T: list of method limitations see
T: https://telepresence.io/reference/methods.html
T: Volumes are rooted at $TELEPRESENCE_ROOT. See
T: https://telepresence.io/howto/volumes.html for details.
T: Starting network proxy to cluster by swapping out Deployment hello-
T: world with a proxy
T: Forwarding remote port 9090 to local port 9090.

T: Connected. Flushing DNS cache.
T: Setup complete. Launching your command.
Serving HTTP on 0.0.0.0 port 9090 (http://0.0.0.0:9090/) ...
```

这里有个疑问，这个把远端 9090 端口映射到本地 9090是什么意思？

`teletepresence --help`看帮助，说是把 Deployment 的端口映射什么什么的，没明白，deployment 管端口什么事，那不 service 的事情吗？

对，其实就是 service的 port 端口（非 targetPort/NodePort）。


```
hugo@zack:/Users/hugo $ kubectl get svc
NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
hello-world   NodePort    10.101.139.40   <none>        8000:32293/TCP   3h31m

hugo@zack:/tmp/telepresence-test $  telepresence --run curl http://10.101.139.40:8000
...
Hello, world!
...
```
注意这里用的是 service_ip和 port（区分 targePort/nodePort）。


上述的操作是替换个新的 deployment。

但是想下我们如果要开发怎样的？

如果我们是要开发一个的功能，如一个新的 web 服务。我们会怎样做？

首先会先写上代码：

```
# helloworld.py
#
#!/usr/bin/env python3
from http.server import BaseHTTPRequestHandler, HTTPServer
class RequestHandler(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.send_header('Content-type', 'text/plain')
        self.end_headers()
        self.wfile.write(b"Hello, world!\n")
        return
httpd = HTTPServer(('', 8080), RequestHandler)
httpd.serve_forever()
```
然后把这个代码跑起来，它是个 web 服务，要放在 cluster 集群中，跟其他服务进行交互。

那么就要部署一个新的服务让它监听到本地的流量。

在 cluster 集群中用一个新的 Deployment，用 teletepresence让它接管本地的 8080 端口的数据：

```
localhost$ telepresence --new-deployment hello-world2 --expose 8080

```

如果是直接替换 cluster 上面本来就有的 Deployment 就用 swapdeployment。

然后在本地把这个 web 跑起来：

```
localhost$ python3 helloworld.py

```

然后就可以在这个 k8s 集群上看到这个 hello-world2 的服务了，然后可以新建个容器，在窗口里面直接访问这个 serviceIP:port 就可以访问到数据了。如下：

```
localhost$ kubectl --restart=Never run -i -t --image=alpine console /bin/sh
kubernetes# wget -O - -q http://hello-world:8080/
Hello, world!
```
但是在 minikube 中，创建pod 来验证有点问题，一直卡着不动。

原因为：minikube ssh 进入之后，发现无法联网，关闭 minikube,配置代理，启动 minikube，minikube ssh 进入后，docker pull alipine。

然后就可以进入 Pod了。
这个是进入 pod 的执行结果：
```
/ # wget -O - -q http://10.100.138.48:8000
Hello, world!
```

但是 minikube 有点特殊，只要进入 minikube ssh 进入后，可以直接获取能用 wget 获取到数据。
但是直接 Ping 这个 service ip 却 ping 不通
```
$ wget -O - -q http://10.100.138.48:8000
Hello, world!

$ ping 10.100.138.48
--- 10.100.138.48 ping statistics ---
1 packets transmitted, 0 packets received, 100% packet loss
$
```
