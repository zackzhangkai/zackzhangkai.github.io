---
layout: post
title:  kubernetes services
date:   2018-04-26 01:08:00 +0800
categories: document
tag:
  - kubernetes

---

* content
{:toc}


# 概述
kubernetes pods很容易消亡，快速创建和销毁，且不可复活。ReplicationControllers会动态的创建和销毁pods（当scale up/down或是rolling updates时）。每个pod拥有自己的IP，但不固定。这就导致一个问题，如果在集群内，backends pods为frontend pods提供功能时，如何能保证frontend正确连接到backend

进入到services内部，kubernetes service就是一系列pods的抽象和一个可以进入它的政策（有时称它为微服务），一个service的一系列pods由Label Selector标识

在下面的例子中，将有一个由3个replicas组成的backend ，这些replicas都是可变的，frontal并不关心backend用哪个。

# 定义一个service
在kubernetes中一个service就是一个REST对象,这跟pod一样。跟其他所有REST对象一样，定义一个Service可通过POST数据到apiserver中去创建一个新实例。现假设你有一系列pods，他们都暴露9376端口，并且有一个label为“app=MyApp”

```
kind: Service
apiVersion: v1
metadata:
  name: my-service
spec:
  selector:
    app: MyApp
  ports:
  - protocol: TCP
    port: 80
    targetPort: 9376
```
上面这个配置文件定义了这些信息：创建一个名为my-service的service，每个pod上tcp port 9376，lalbe: app=MyApp。该 service同时还被绑定了一个IP：9376(有时叫他clusterIP),它能被service proxy使用。service的selector将被连续的评估，且数据也将被post到名为my-service的端点上。注意：service可将一个入口端口映射到目标端口。

>service一般都需要有selectors，除了少数情况

# Virtual IPs and service proxies
在k8s的每个node节点上都会有一个kube-proxy,kube-proxy主要是保证service的虚IP格式而非ExternalName.在kubernetes v1.0版本中，service是4层的结构，proxy在用户层。从v1.1开始， ingress API被加入第七层（HTTP）,iptables proxy同样也加入了，并且从v1.2开始变成了默认的操作模式。从v1.8开始ipvs proxy也增加了。

## Proxy-mode: userspace
在这个模式下，kube-proxy监视kubenetes master对service和endpoints对象的增加和移除。对每个service而言，它将安装iptables rules来捕捉到service 的clusterIP(虚拟的)和port流量，并将它重定向到backend的服务上。很显然iptables不用在用户空间和内核空间来回切换，它比userspace proxy更快更值得信任。会随机选择一个backend pod来安装iptables rule。然而不同于user proxy，如果iptable proxier不能自动重试一另一个pod，如果被选择的这个pod无法响应。 因此它依赖于[readiness probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/#defining-readiness-probes)
<img src="{{ '/styles/images/services-iptables-overview.svg' | prepend: site.baseurl }}" alt="" width="810" />
>注意上图中，service ip 就是cluster ip

## Proxy-mode: ipvs
在这种模式下，kube-proxy监视servcies和endpoints.名为"netlink"的接中会创建ipvs rules,并且周期性的同步这些规则，保证ipvs的状态。当有服务加入时，流量会被自动的负载到其中的一个pod上。与iptables类似，ipvs依赖netfilter hook,但它工作在内核层。这意味这它在同步规则时会更快。另ipvs有更多的LB选择，如rr(round-robin),lc(least connection),dh(destination hashing),sh(ource hashing),sed(shortest expected delay),nq(never queue)。
_ipvs mode需要kube proxy之前安装好。当kube-proxy以ipvs proxy模式启动时，才会成功，否则它将退回到iptables proxy模式_

<img src="{{ '/styles/images/services-ipvs-overview.svg' | prepend: site.baseurl }}" alt="" width="810" />

In any of these proxy model, any traffic bound for the Service’s IP:Port is proxied to an appropriate backend without the clients knowing anything about Kubernetes or Services or Pods. Client-IP based session affinity can be selected by setting service.spec.sessionAffinity to “ClientIP” (the default is “None”), and you can set the max session sticky time by setting the field service.spec.sessionAffinityConfig.clientIP.timeoutSeconds if you have already set service.spec.sessionAffinity to “ClientIP” (the default is “10800”).

# Multi-Port Services
当使用多个端口时，需要给每个端口一个名字，这样endpoint才不会混淆。
```
kind: Service
apiVersion: v1
metadata:
  name: my-service
spec:
  selector:
    app: MyApp
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 9376
  - name: https
    protocol: TCP
    port: 443
    targetPort: 9377
```

# Discovering services
# DNS
这是可选的，但是强烈建议安装的[add-on](https://kubernetes.io/docs/concepts/cluster-administration/addons/)，

For example, if you have a Service called "my-service" in Kubernetes Namespace "my-ns" a DNS record for "my-service.my-ns" is created. Pods which exist in the "my-ns" Namespace should be able to find it by simply doing a name lookup for "my-service". Pods

which exist in other Namespaces must qualify the name as "my-service.my-ns". The result of these name lookups is the cluster IP.

The Kubernetes DNS server is the only way to access services of type ExternalName

# Headless services
Sometimes you don’t need or want load-balancing and a single service IP. In this case, you can create “headless” services by specifying "None" for the cluster IP (spec.clusterIP).

This option allows developers to reduce coupling to the Kubernetes system by allowing them freedom to do discovery their own way. Applications can still use a self-registration pattern and adapters for other discovery systems could easily be built upon this API.

For such Services, a cluster IP is not allocated, kube-proxy does not handle these services, and there is no load balancing or proxying done by the platform for them. How DNS is automatically configured depends on whether the service has selectors defined

[更多](https://kubernetes.io/docs/concepts/services-networking/service/)

## With selectors
## Without selectors

# Publishing services - service types

某些应用需要将ip暴露到外面，Type可以来定义它的类型，有以下几种：

## ClusterIP
这个是默认的，只能在cluster内部暴露

## NodePort
Exposes the service on each Node’s IP at a static port (the NodePort). A ClusterIP service, to which the NodePort service will route, is automatically created. You’ll be able to contact the NodePort service, from outside the cluster, by requesting <NodeIP>:<NodePort>.

## LoadBalancer
Exposes the service externally using a cloud provider’s load balancer.

```
kind: Service
apiVersion: v1
metadata:
  name: my-service
spec:
  selector:
    app: MyApp
  ports:
  - protocol: TCP
    port: 80
    targetPort: 9376
  clusterIP: 10.0.171.239
  loadBalancerIP: 78.11.24.19
  type: LoadBalancer
status:
  loadBalancer:
    ingress:
    - ip: 146.148.47.155
```

## ExternalName
Maps the service to the contents of the externalName field (e.g. foo.bar.example.com), by returning a CNAME record with its value. No proxying of any kind is set up. This requires version 1.7 or higher of kube-dns
