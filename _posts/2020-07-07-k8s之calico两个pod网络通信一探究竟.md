---
layout: post
published: true
title:  k8s之calico两个pod网络通信一探究竟
categories: [ k8s ]
tags: [ 网络 ]
---
* content
{:toc}



# 通信过程

两个pod相ping，以cordns的pod为例
```
[root@node1 ~]# kubectl -n kube-system get po -o wide|grep coredns
coredns-79c6f6447f-6f9xp                       1/1     Running   0          44h   10.233.74.65   node4   <none>           <none>
coredns-79c6f6447f-hp7ng                       1/1     Running   0          44h   10.233.71.1    node3   <none>           <none>

[root@node3 ~]# nsenter -n -t 30319 ping 10.233.74.65
PING 10.233.74.65 (10.233.74.65) 56(84) bytes of data.
64 bytes from 10.233.74.65: icmp_seq=1 ttl=62 time=0.742 ms

[root@node3 ~]# nsenter -n -t 30319 ip r
default via 169.254.1.1 dev eth0
169.254.1.1 dev eth0 scope link

[root@node3 ~]# nsenter -n -t 30319 ip r get 10.233.74.65
10.233.74.65 via 169.254.1.1 dev eth0 src 10.233.71.1
```
通过命令我们知道，现在是将这个包通过eth0发到了169.254.1.1上。一般下一跳的地址跟当前网络应该是同一个子网地址。然后再在这个子网中发送arp来获取对应的mac地址。然后跟上面场景一样，将数据包的目的mac改为该网关的mac。  
现在看下它的mac地址：
```
[root@node3 ~]# nsenter -n -t 30319 ip n | grep 169.254.1.1
169.254.1.1 dev eth0 lladdr ee:ee:ee:ee:ee:ee REACHABLE
```
此时这个抓下eth0的包看下：
```
[root@node3 ~]#  nsenter -n -t 30319 tcpdump -i eth0 icmp -qenc 4
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
13:51:46.738340 2e:b2:a2:2a:a4:02 > ee:ee:ee:ee:ee:ee, IPv4, length 98: 10.233.71.1 > 10.233.74.65: ICMP echo request, id 32418, seq 19, length 64
13:51:46.738831 ee:ee:ee:ee:ee:ee > 2e:b2:a2:2a:a4:02, IPv4, length 98: 10.233.74.65 > 10.233.71.1: ICMP echo reply, id 32418, seq 19, length 64
13:51:47.738303 2e:b2:a2:2a:a4:02 > ee:ee:ee:ee:ee:ee, IPv4, length 98: 10.233.71.1 > 10.233.74.65: ICMP echo request, id 32418, seq 20, length 64
13:51:47.738708 ee:ee:ee:ee:ee:ee > 2e:b2:a2:2a:a4:02, IPv4, length 98: 10.233.74.65 > 10.233.71.1: ICMP echo reply, id 32418, seq 20, length 64
```
此时可以看到目的MAC确实变成了ee:ee这种。源mac应该是eth0的mac，验证下：

```
[root@node3 ~]#  nsenter -n -t 30319 ip ad show eth0
4: eth0@if10: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1440 qdisc noqueue state UP group default
    link/ether 2e:b2:a2:2a:a4:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.233.71.1/32 scope global eth0
       valid_lft forever preferred_lft forever
```
符合预期。

**总结1： 当一台机器要访问网关的时候，首先会通过 ARP 获得网关的 MAC 地址，然后将目标 MAC 变为网关的 MAC，而网关的 IP 地址不会在任何网络包头里面出现。**

因此配置网关的最重要作用就是通过这个网关的IP能拿到它的MAC地址，好把包送过去。即 `ip n| grep <GateWayIP>`获得到dst MAC ID。

事实上，在容器中也Ping不通这个网关IP，但是不要紧，重要的是通过这个IP能拿到MAC，它就完成了任务。另外这个网关IP是calico给它分配的，没有任何一个网卡有这个IP, arp也是calico配的。

由于eth0是veth pair，因此找下它的对端：
```
[root@node3 ~]# nsenter -n -t 30319 ethtool -S eth0
NIC statistics:
     peer_ifindex: 10
```
在node上看下：
```
ip ad
10: calidd7606fae3a@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1440 qdisc noqueue state UP group default
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet6 fe80::ecee:eeff:feee:eeee/64 scope link
       valid_lft forever preferred_lft forever
```

此时可以看到这个网卡的地址就是目的mac。根据通信原则，如果目的mac是自己，而目的IP不是自己，那么就会在当前命名空间路由。  
看下路由：
```
[root@node3 ~]# ip r get 10.233.74.65
10.233.74.65 via 192.168.0.5 dev tunl0 src 10.233.71.0
    cache expires 542sec mtu 1440
```
可以看到这个包下一跳是192.168.0.5，从tunl0发出去，～～要求把源地址改为10.233.71.0～～。

在node3上抓下包
```
[root@node3 ~]# tcpdump -i any icmp and host 10.233.74.65 -qenc 4
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on any, link-type LINUX_SLL (Linux cooked), capture size 262144 bytes
13:36:48.817330  In 2e:b2:a2:2a:a4:02 10.233.71.1 > 10.233.74.65: ICMP echo request, id 13311, seq 55, length 64
13:36:48.817379 Out 10.233.71.1 > 10.233.74.65: ICMP echo request, id 13311, seq 55, length 64
13:36:48.817893  In 10.233.74.65 > 10.233.71.1: ICMP echo reply, id 13311, seq 55, length 64
13:36:48.817931 Out ee:ee:ee:ee:ee:ee 10.233.74.65 > 10.233.71.1: ICMP echo reply, id 13311, seq 55, length 64
```
通过上面的包发现了有些有 `In` `Out`的这些包有MAC ID，再看，发现每个seq对应的一个包，有四个记录。这个原因是本机路由了一次，先是把包发到了calicoxxxx这个上（in）；这个网卡发现不是自己的，然后将这个包发出来，在这个主机上路由一下(out)；路由的结果还是将这个包发到本机的另一个网卡tunl0上（in）。这个网卡接收到后，通过arp获取下一跳ip（192.168.0.5）的mac后，改变目的mac，把它发出去（out）。


可以看到源ip并没有发生变化，但是它的路由中有个`src`字段，经过查资料发现 **`src`字段仅只有包是从本主机出去的时候，才会加的源IP，而不能改变已经有了源IP的数据包** 。

通过上面的路由知道这个包发到了 `192.168.0.5`上，这个目的ip pod的宿主机node4。

这里有个问题，为什么要有这个路由？
```
10.233.74.65 via 192.168.0.5 dev tunl0 src 10.233.71.0
```
一般的如果是指定从哪个dev出去，一般的这个dev与目的网络应该是同一个子网，但是从这个接口无法直接通这个IP
```
[root@node3 ~]# ping 192.168.0.5 -I tunl0
PING 192.168.0.5 (192.168.0.5) from 10.233.71.0 tunl0: 56(84) bytes of data.
^C
--- 192.168.0.5 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 999ms
```

现在看下tunl0这个dev
```
[root@node3 ~]# ip r | grep tunl0
10.233.74.64/26 via 192.168.0.5 dev tunl0 proto bird onlink

[root@node3 ~]# ip -d link show tunl0
9: tunl0@NONE: <NOARP,UP,LOWER_UP> mtu 1440 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/ipip 0.0.0.0 brd 0.0.0.0 promiscuity 0
    ipip remote any local any ttl inherit nopmtudisc addrgenmode eui64 numtxqueues 1 numrxqueues 1 gso_max_size 65536 gso_max_segs 65535
...
```

猜测，可能跟这个tunl0设备有关，因为calico用的是ipip，即是ip in ip，就是在ip的基础上再封装一层。所以经过这个包后，这个数据包应该经过了一层封装。

看下calico的配置
```
[root@node1 ~]# kubectl -n kube-system get cm calico-config -o yaml | less

apiVersion: v1
data:
  calico_backend: bird
  cni_network_config: |-
    {
      "name": "k8s-pod-network",
      "cniVersion": "0.3.1",
      "plugins": [
        {
          "type": "calico",
          "log_level": "info",
          "datastore_type": "kubernetes",
          "nodename": "__KUBERNETES_NODE_NAME__",
          "mtu": __CNI_MTU__,
          "ipam": {
              "type": "calico-ipam"
          },
          "policy": {
              "type": "k8s"
          },
          "kubernetes": {
              "kubeconfig": "__KUBECONFIG_FILEPATH__"
          }
        },
        {
          "type": "portmap",
          "snat": true,
          "capabilities": {"portMappings": true}
        },
        {
          "type": "bandwidth",
          "capabilities": {"bandwidth": true}
        }
      ]
    }
  typha_service_name: none
  veth_mtu: "1440"
kind: ConfigMap
...
```
并没有看出什么异常，抓tunl0的包看下：
```
[root@node3 ~]# tcpdump -i tunl0 icmp and host 10.233.74.65  -qenc 10
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on tunl0, link-type RAW (Raw IP), capture size 262144 bytes
15:34:42.703307 ip: 10.233.71.1 > 10.233.74.65: ICMP echo request, id 19745, seq 456, length 64
15:34:42.703916 ip: 10.233.74.65 > 10.233.71.1: ICMP echo reply, id 19745, seq 456, length 64
15:34:43.703279 ip: 10.233.71.1 > 10.233.74.65: ICMP echo request, id 19745, seq 457, length 64
15:34:43.703755 ip: 10.233.74.65 > 10.233.71.1: ICMP echo reply, id 19745, seq 457, length 64
15:34:44.703271 ip: 10.233.71.1 > 10.233.74.65: ICMP echo request, id 19745, seq 458, length 64
15:34:44.703749 ip: 10.233.74.65 > 10.233.71.1: ICMP echo reply, id 19745, seq 458, length 64
15:34:45.703198 ip: 10.233.71.1 > 10.233.74.65: ICMP echo request, id 19745, seq 459, length 64
15:34:45.703582 ip: 10.233.74.65 > 10.233.71.1: ICMP echo reply, id 19745, seq 459, length 64
15:34:46.703265 ip: 10.233.71.1 > 10.233.74.65: ICMP echo request, id 19745, seq 460, length 64
15:34:46.703600 ip: 10.233.74.65 > 10.233.71.1: ICMP echo reply, id 19745, seq 460, length 64
10 packets captured
10 packets received by filter
0 packets dropped by kernel
```
通过tunl0肯定无法直接如192.168.0.5这个Node的eth0 IP直接通信，肯定还是用eth0来通信的。
抓下eth0的包：
```
[root@node3 ~]# tcpdump -i eth0 host 192.168.0.5 -qenc 10
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
15:41:34.770387 52:54:22:dd:df:0d > 52:54:22:0b:2e:bd, IPv4, length 118: 192.168.0.4 > 192.168.0.5: 10.233.71.1 > 10.233.74.65: ICMP echo request, id 19745, seq 868, length 64 (ipip-proto-4)
15:41:34.771372 52:54:22:0b:2e:bd > 52:54:22:dd:df:0d, IPv4, length 118: 192.168.0.5 > 192.168.0.4: 10.233.74.65 > 10.233.71.1: ICMP echo reply, id 19745, seq 868, length 64 (ipip-proto-4)
15:41:35.007235 52:54:22:0b:2e:bd > 52:54:22:dd:df:0d, IPv4, length 86: 192.168.0.5 > 192.168.0.4: 10.233.74.65.domain > 10.233.71.0.40130: tcp 0 (ipip-proto-4)
15:41:35.047284 52:54:22:dd:df:0d > 52:54:22:0b:2e:bd, IPv4, length 86: 192.168.0.4 > 192.168.0.5: 10.233.71.0.40130 > 10.233.74.65.domain: tcp 0 (ipip-proto-4)
15:41:35.771665 52:54:22:dd:df:0d > 52:54:22:0b:2e:bd, IPv4, length 118: 192.168.0.4 > 192.168.0.5: 10.233.71.1 > 10.233.74.65: ICMP echo request, id 19745, seq 869, length 64 (ipip-proto-4)
15:41:35.772121 52:54:22:0b:2e:bd > 52:54:22:dd:df:0d, IPv4, length 118: 192.168.0.5 > 192.168.0.4: 10.233.74.65 > 10.233.71.1: ICMP echo reply, id 19745, seq 869, length 64 (ipip-proto-4)
15:41:36.023596 52:54:22:dd:df:0d > 52:54:22:0b:2e:bd, IPv4, length 86: 192.168.0.4 > 192.168.0.5: 10.233.71.1.domain > 10.233.74.64.49148: tcp 0 (ipip-proto-4)
15:41:36.064699 52:54:22:0b:2e:bd > 52:54:22:dd:df:0d, IPv4, length 86: 192.168.0.5 > 192.168.0.4: 10.233.74.64.49148 > 10.233.71.1.domain: tcp 0 (ipip-proto-4)
```
发现很多结尾还有ipip-proto0-4的标签。然后192.168.0.5 node4收到这个包后，发现还有ipip，会先把包抓开，然后就还原了原数据包，然后再路由进pod，这就是完整的步骤。
```
[root@node4 ~]# ip r get 10.233.74.65
10.233.74.65 dev cali9a8d0827254 src 192.168.0.5  #这个cali同样是veth，另一端连着容器，这样包就进了容器里面。回程同理。
```

补充，calico有两种工作模式，bpg或是ipip，Ipip会让包走tunl0，如果是bpg，包直接走eth0。[参考](https://segmentfault.com/a/1190000016565044)


# 容器的每个IP都一个独立的子网：
```
[root@node3 ~]# nsenter -n -t 30319 ip ad                                                      │#       disabled;
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000    │#       local as 65000;
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00                                      │#       multihop;
    inet 127.0.0.1/8 scope host lo                                                             │#       rr client;
       valid_lft forever preferred_lft forever                                                 │#       rr cluster id 1.0.0.1;
2: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN group default qlen 1000                  │#}
    link/ipip 0.0.0.0 brd 0.0.0.0                                                              │#
4: eth0@if10: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1440 qdisc noqueue state UP group default  │#protocol bgp rr_abcd from rr_client {
    link/ether 2e:b2:a2:2a:a4:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0                          │#       neighbor 10.1.4.7 as 65000;
    inet 10.233.71.1/32 scope global eth0                                                      │#}
       valid_lft forever preferred_lft forever
```
看到容器中的Ip是 10.233.71.1/32，每个IP是一个独立的子网，访问任何其他的IP都需要路由，走到网关。

# 路由配置组件 Felix

就像有三台物理机，两两之间都需要配置路由，每台物理机上对外的路由就有两条。如果有六台物理机，则每台物理机上对外的路由就有五条。新加入一个节点，需要通知每一台物理机添加一条路由。这还是在物理机之间，一台物理机上，每创建一个容器，也需要多配置一条指向这个容器的路由。如此复杂，肯定不能手动配置，需要每台物理机上有一个 agent，当创建和删除容器的时候，自动做这件事情。这个 agent 在 Calico 中称为 Felix。

# 路由广播组件 BGP Speaker
在 Calico 中，每个 Node 上运行一个软件 BIRD，作为 BGP 的客户端，或者叫作 BGP Speaker，将“如何到达我这个 Node，访问我这个 Node 上的容器”的路由信息广播出去。所有 Node 上的 BGP Speaker 都互相建立连接，就形成了全互连的情况，这样每当路由有所变化的时候，所有节点就都能够收到了。


参考网易刘超的文章 [calico](https://time.geekbang.org/column/article/11940)

# 附：
其实ipip是可以抓到内层包的：
```
[root@master1 logtest]# tcpdump -i eth0 proto 0x04
06:46:16.719212 IP (tos 0x0, ttl 64, id 16045, offset 0, flags [DF], proto IPIP (4), length 80)
    192.168.11.154 > 192.168.11.201: IP (tos 0x0, ttl 64, id 3404, offset 0, flags [DF], proto TCP (6), length 60)
    10.233.118.3.36878 > 10.233.98.28.53: Flags [S], cksum 0xf936 (correct), seq 617938112, win 43690, options [mss 65495,sackOK,TS val 2165957
84 ecr 0,nop,wscale 7], length 0
```
看这个包，内层IP和外层IP。

为什么是 proto 0x04?

因为IPIP对应的这个字段是16进制，因些要加0x04。可以通过抓所有包，然后grep过滤`IPIP`,可以看到IPIP的包后面的括号是有个4,然后就知道是这个值。

关于包的详解，及每个字段表示的意思，[参考](https://www.cnblogs.com/ggjucheng/archive/2012/02/02/2335495.html)

在抓包的时候如果开了网卡 offload 时，经常看到有checksum的值是incorrect，但是返回包又是正常的，原因是tcpdump是在数据包被发送给网卡之前捕捉数据包的，此时它不会看到正确的checksum，因为此时尚未进行计算（因为checksum已经卸载到网卡，此时这个checksum字段会被填写为0）只有经过网卡后，checksum才会被正确设置。这也就导致了抓包工具提示checksum错误的原因。 [参考](https://huataihuang.gitbooks.io/cloud-atlas/network/packet_analysis/tcpdump/udp_tcp_checksum_errors_from_tcpdump_nic_hardware_offloading.html)

关闭checksum offload

```
ethtool -K ethX rx off tx off  #这其实是临时生效的方法，持久化需要在网卡的配置文件中/etc/sysconfig/ethxxx
```
为什么需要这个东西？

如果关闭这个功能，当一个大包来的时候，会由操作系统执行分片处理，消耗系统资源；开启后，大包进来后，由网卡执行卸载，节省系统资源。

tcpdump抓到的包，如何看这个包是否是分片的？

```
07:28:35.650067 IP (tos 0x10, ttl 64, id 9908, offset 0, flags [DF], proto TCP (6), length 2848)
    192.168.11.201.22 > 171.113.240.2.25935: Flags [.], cksum 0x72f8 (incorrect -> 0x146d), seq 283258:286054, ack 217, win 45, options [nop,nop,TS val 146281809 ecr 824068609], length 2796
```
如这个包中，flags [DF]  就是未分片。[MF]是分片，注意是小写`flags`。包后面还有个Flags，这个是tcp的Header中的，它是三次握手四次握手中的东西 ` S S.  .  P`，分别表示`seq 、seq+ack 、ack 、（建立连接）push http包`

如何看重传的包？

```
tcpdump -n -v 'tcp[tcpflags] & (tcp-rst) != 0'
```