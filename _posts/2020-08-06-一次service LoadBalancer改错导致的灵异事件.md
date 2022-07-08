---
layout: post
published: true
title:  一次service LoadBalancer改错导致的灵异事件
categories: [ k8s ]
tags: [troubleshooting]
---
* content
{:toc}


# 问题产生背景

动机：我想给ks-apiserver 这个service 改成LoadBalancer的，然后我这样做的：

```bash
# kubectl edit svc ks-apiserver -n kubesphere-system
...
spec:
  clusterIP: 10.233.7.171
  externalTrafficPolicy: Cluster
  ports:
  - nodePort: 32302
    port: 80
    protocol: TCP
    targetPort: 9090
  selector:
    app: ks-apiserver
    tier: backend
    version: v3.0.0
  sessionAffinity: None
  type: LoadBalancer  # 由ClusterIP改成LoadBalancerIP
  externalIPs:  # 新加这个，写上外部IP，否则service的exteralIPs会一直pending
    - 192.168.11.250  # 这个是lb的私网IP
    - x.x.x.x  # 这个写的是lb对应的EIP
...
```

好了，好戏来了，等我保存之后，然后执行kubectl 这个命令已经不管用了。

```bash
[root@master1 logtest]# kubectl get po
The connection to the server lb.kubesphere.local:6443 was refused - did you specify the right host or port?
```

what's wrong????  （一脸黑线，此处应有黑人一脸懵逼的表情。。）

# 分析与排查

冷静下下来分析下，这个命令本质上是调kube-apiserver，只有两种可能，要么是IP不通，要么是端口有问题。

curl 下这个接口

```bash
# curl -k https://lb.kubesphere.local:6443 # 发现端口时通时不通
# nc -vz 192.168.11.250 6443  # 端口时通时不通
# ping 192.168.11.250  # 正常Ping通
```

登陆云主机管理后台，看Lb的监听器，都显示正常
从页面进入lb的虚机，发现6443端口也是正常监听（因为是tcp监听器），此时能正常curl通端口

`curl -k https://192.168.11.250:6443` 正常。

这里就有点意思了，都正常，为啥会这样呢？

想一下，问题应该在Lb身上，如果不经过lb是不是就正常了？

```bash
# cat /etc/hosts
192.168.11.250  lb.kubesphere.local  # 把Lb的地址，改成自已的 192.168.11.201后，再次使用
# kubectl后正常
```

看来果然是lb的问题。

直接ssh lb的IP试下：

`ssh 192.168.11.250` 发现会随机在集群中选择一个主机

![](/styles/images/troubleshooting01.png)

这里就有点意思了。更加不理解了，此时有一万个冲动，把集群重装，觉得集群出了问题，修不好了，太奇怪了，完全不理解了。不过我相信，一切问题皆是有原因的，没有无缘无故的问题。

此时到处询问了别人这个奇怪的现象，也都表示没见过，不知道，IP冲突换个IP之内的。但是检查这个私有网络的IP发现并没有冲突。算了，还是靠自己。

想到了抓包，看这个包是不是正常走到了lb上。

```bash
# 先在 lb的 instance上 抓包，然后在master1上执行curl或是nc，看下目的端口的包，Lb instance是否能正常收到
tcpdump -i any src host 192.168.11.201 and dst port 6443
```

发现直接可以抓到很多包，想一下，master1在一直跟lb通信，因为master1是lb的一个后端，它们肯定会一直通信，因此刚才抓的包并不能说明任何问题，因为并不能区分出哪个包是刚才发出的。

既然在server端抓不到想要的包，索性在客户端看下这个包有没有正常发出去。

在master1上抓所有的发往Lb的包。

```bash
tcpdump -i any host 192.168.11.250 port 6443 -w master1.pcap
```

然后，在master1上通过nc/curl来发一次请求。

终止抓包，把包拷到本地，用wireshark分析：

```bash
scp -P 30012 root@kk1:/root/master1.pcap /tmp
open /Applications/Wireshark.app /tmp/master1.pcap
```

在wireshark上过滤下包，wireshark的语法，直接点击输入框右边的下三角，有很多示例，找一个套下就出来了。

![troubleshooting2.png](/styles/images/troubleshooting2.png)

看到客户端发起一个包后，服务端直接RST了。

![](/styles/images/troubleshooting3.png)

上网查了下，RST的原因是服务端这个端口不存在，直接拒绝。

更纳闷了，lb的instance上面，明明看到端口是监听的！！！

莫非这个IP不是那个IP?

直接把Lb关机，发现ssh lb的ip地址，还是能随机跳到任意一台node上面。

检查该node的IP，用的命令是

```bash
[root@node2 ~]# hostname -I
192.168.11.3 172.17.0.1 10.233.96.0
```

发现并没有lb的ip，为什么能跳过来呢？

>其实此时，看某个节点有没有某个ip，不能使用这种方式。应该用 `ip ad|grep xxx` 来过滤，就可以发现，这个IP是kube-ipvs0上的一个IP。（但是当时由于使用上面的命令，因此暂时还没有发现）

先ssh lb的ip ,然后在master1上看下这个包的状态：

![](/styles/images/troubleshooting4.png))

看到包确实有连接。

以为是ipvs把这个LB 的IP做了负载均衡，转发到了所有的后端，但是通过`ipvsadm -Sn|grep 192.168.11.250` ，发现并没有。

想一下包是怎么走的？

ssh 192.168.11.250到底发生了什么？

ssh到一个IP地址，本质上把包发往这个目的地址，首先要看这个IP的mac地址，要先发起arp，看这个IP对应的mac是多少。因此看下它的mac

```bash
ip n|grep 192.168.11.250 # 发现有一个mac地址
```

抓包看下arp

```bash
tcpdump -i any arp and host 192.168.11.250  -nn
```

![](/styles/images/troubleshooting5.jpg)

此时真相大白了，这个Ip有很多mac，说明这个ip确实重复了。

再想一下，即使这个包送到了对应的这个mac的主机上，但是Ip不对，也只能是路由，但是为什么会直接连上了呢？只有一种可能，那就是这个主机上有这个Ip地址跟mac地址。

直接去node2上看 ip a | grep 192.168.11.250看下，发现果然。。。

# 总结：

好了，总结下：因为刚才在修改ks-apiserver的service类型为LoadBalancer时，添加了exterternalIPs中有 192.168.11.250，因此，会在所有的主机的kube-ipvs0这个网卡加上它的IP。由于是ipvs的，所有的serviceIP都是能直接Ping通的。因此，curl/nc发请求时，并没有走到Lb上，而是走到了随机的一个节点上，当这个请求到了master上就会通，到了worker就不通。这就是为什么时通时不通的原因。

kubectl 不能用的原因也可以解释了，因为 kubeconfig文件中请求是指向LB的IP地址的。

由于把ks-apiserver service直接删除后，问题也并没有解决。直接在所有的节点kube-ipvs0上把这个IP删掉，也没有解决。因此重装了环境。其实，如果直接保留这个service，把这个externalIPs去掉，或是改成clusterIP类型的，问题是不是就解决了？有待验证。（验证，删除这个servcie，kube-proxy上面的ip删除了，如果不删除这个service，直接把类型改成clusterIP，或是去掉这个externalIP，Kube-proxy上面的Ip也去掉了。）

说明：这个问题造成的原因是对如何把一个服务暴露成loadbalancer不熟造成的。

问了同事，同事说这个值不用设置，service lb 的controller会自动给它申请一个lb的地址。感觉他们也没有理解，我说我想自己指定Lb的eip，他们也没有答上来，有的说加annotation，有的说不用指定，它会自动跟你生成一个。自动生成，其实是qke确实会跟你生成。但是跟我的需求完全不一样。

好了，不纠结了，正确的设置 LoadBalancer service 需要注意的是：这个externalIPs设置中的IP不能跟 kubeconfig中设置的lb地址冲突，否则有问题。

# 附一个正确LoadBalancer svc的完整yaml: 

```bash
[root@master1 ~]# kubectl -n kubesphere-system get svc ks-apiserver -oyaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{"kubernetes.io/created-by":"kubesphere.io/ks-apiserver"},"labels":{"app":"ks-apiserver","tier":"backend","version":"v3.0.0"},"name":"ks-apiserver","namespace":"kubesphere-system"},"spec":{"ports":[{"port":80,"protocol":"TCP","targetPort":9090}],"selector":{"app":"ks-apiserver","tier":"backend","version":"v3.0.0"},"type":"ClusterIP"}}
    kubernetes.io/created-by: kubesphere.io/ks-apiserver
  creationTimestamp: "2020-08-06T04:54:13Z"
  labels:
    app: ks-apiserver
    tier: backend
    version: v3.0.0
  name: ks-apiserver
  namespace: kubesphere-system
  resourceVersion: "391607"
  selfLink: /api/v1/namespaces/kubesphere-system/services/ks-apiserver
  uid: 9dfcd105-4bbc-41dd-b422-7bee7e9ca1b4
spec:
  clusterIP: 10.233.7.171
  externalIPs:
  - 139.198.19.140
  externalTrafficPolicy: Cluster
  ports:
  - nodePort: 32302
    port: 80
    protocol: TCP
    targetPort: 9090
  selector:
    app: ks-apiserver
    tier: backend
    version: v3.0.0
  sessionAffinity: None
  type: LoadBalancer
status:
  loadBalancer: {}
```