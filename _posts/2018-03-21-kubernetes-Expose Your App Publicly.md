---
layout: post
title:  kubernetes-Exopse Your App Publicly
date:   2018-03-21 01:08:00 +0800
categories: document
tag:
  - kubernetes
---

* content
{:toc}

### Kubernetes Services
Kubernetes Pods都是有生命周期的，若一个node死掉，运行在上面的pods都会同时丢失。此时ReplicationController会按照cluster预设的状态，创建新的pods来维持应用正常运行。Service是一个抽象，定义了一组pods和可以进入它们的policy,让pod销毁或是复制，而不会影响应用。Services通过lables和selectors匹配一组pods。service通过LableSelector定义一组pods。每个Pod都有一个惟一的IP，但是这些IP在没有服务的情况下不会暴露在外。服务在ServiceSpec中指定以下<type>来暴露服务，接收流量。
+ ClusterIP（默认）：暴露cluster内部IP，这种类型使得服务只能在集群中访问
+ NodePort：通过NAT在每个选择的NODE上暴露相同的端口，使得服务能在集群外部访问：<NodeIP>:<NodePort>
+ LoadBalancer
+ ExternalName: 通过一个任意的名字（在spec中指定externalName）返回一个CNAME记录来暴露服务，无需Proxy，版本要求1.7+，或kube-dns

注意：spec在没有定义selector时同样可以创建一个配对应的Endpoints对象。这个要求用户手动的映射服务到指定的endpoints上。还有一种情况就是你使用了type:ExternalName


<img src="{{ '/styles/images/kubernetes-service.png' | prepend: site.baseurl }}" alt="Service and Lables" width="410" />
<img src="{{ '/styles/images/k8s-service2.png' | prepend: site.baseurl }}" alt="Service and Lables" width="410" />

创建一个service后，有cluster_ip/内部port/外部IP（nodeIP）
```
kubectl expose deployment/kubernetes-bootcamp --type="NodePort" --port 8080
kubectl describe services/kubernetes-bootcamp
```

deployment自动为我们的pod创建了lable，用describe deployment可以看见创建的lable
```
kubectl describe deployment
```
我们可以用kubectl get pods命令加上-l作为参数，后跟lable value来查询这个pod
```
kubectl get pods -l run=kubernetes-bootcamp
```
服务类似
```
kubectl get services -l run=kubernetes-bootcamp
```
获取pod_name
```
export POD_NAME=$(kubectl get pods -o go-template --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}')
echo Name of the Pod: $POD_NAME
curl $(minikube ip):$NODE_PORT
```

给pod加入一个新的lable
```
kubectl lable pod $POD_NAME app=v1
```
同样可以用这个lable
```
kubectl describe pods $POD_NAME
kubectl get pods -l app=v1
```
删除一个服务service
```
kubectl delete service -l run=kubernetes-bootcamp
```
确认这个服务已经不存在
```
kubectl get services
```
此时从外部已经无法访问，但是从内部仍可访问
```
curl $(minikube ip):$NODE_PORT    无法访问
kubectl exec -ti $POD_NAME curl localhost:8080 可以访问
```
