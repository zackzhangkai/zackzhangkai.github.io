---
layout: post
published: true
title:  利用namespace做资源网络资源隔离及组网
categories: [ linux ]
tags: [namespace]
---
* content
{:toc}


# 前言

不想每次都创建虚机，如果能用ns直接把网络环境做个模拟，会简化很多操作，又不会污染当前的虚机环境。

本实验使用的一台宿主机上面创建ns  
宿主机IP: 192.168.0.2/24  
ns的网络 172.30.0.0/16


# 单个ns环境组网
新建一个namespace，让它与宿主机实现网络联通。也可以配置iptables做snat转换后，伪装源Ip后，实现访问外网。

## 与主机环境联通

```bash
#创建namespace
ip netns add ns01

#创建veth pair
ip link add name veth0 type veth peer name veth1

# 配置网卡
ifconfig veth0 172.30.1.2/16 up

# 将veth11放入ns中
ip link set veth1 netns ns1

# 进入ns
ip netns exec ns1 bash

# 配置网卡（现在是在ns中操作）
ifconfig veth1 172.30.2.2/16 up

# 配置路由，另一端的pair充当网关，进行路由。
ip r add default via 172.30.1.2 dev veth1
```

至此已经配置完成，可以看到ns中到网关和宿主机都能通

```
#在ns中现在ping下192.168.0.2宿主机的ip，看下是否能通

[root@node1 ~]# ping 172.30.1.2
PING 172.30.1.2 (172.30.1.2) 56(84) bytes of data.
64 bytes from 172.30.1.2: icmp_seq=1 ttl=64 time=0.136 ms
^C
--- 172.30.1.2 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.136/0.136/0.136/0.000 ms
[root@node1 ~]# ping 192.168.0.2
PING 192.168.0.2 (192.168.0.2) 56(84) bytes of data.
64 bytes from 192.168.0.2: icmp_seq=1 ttl=64 time=0.077 ms
^C
--- 192.168.0.2 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.077/0.077/0.077/0.000 ms

```

如果只是个网络隔离的作用，那么目前已经完成了任务。现在可以在这个ns中，去搭建一些服务，然后本地就可以来访问这个服务。

但是如果想让这个ns的网络环境访问外网呢？

## 让ns网络访问外网

现在ns ping外网是不通的。
```bash
# ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
^C
--- 8.8.8.8 ping statistics ---
1 packets transmitted, 0 received, 100% packet loss, time 0ms
```

**原因是在宿主机上iptables nat表需要做映射**

需要做映射的原因是：包在宿主机上路由后，通过宿主机发往下一跳后，下一跳的网关会做包的过滤。此时需要把源Ip伪装成宿主机的ip，才能通过。

```bash
iptables -t nat -A POSTROUTING -s 172.30.0.0/16  -j SNAT --to 192.168.0.2
```

现在就可以看到已经通了。

```bash
# ns中操作
[root@node1 ~]# ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=117 time=1.63 ms
^C
--- 8.8.8.8 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 1.639/1.639/1.639/0.000 ms
```
有了snat的转换后，域名也可以用了
```bash
[root@node1 ~]# ping www.baidu.com
PING www.wshifen.com (103.235.46.39) 56(84) bytes of data.
64 bytes from 103.235.46.39 (103.235.46.39): icmp_seq=1 ttl=54 time=2.11 ms
^C
--- www.wshifen.com ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 2.118/2.118/2.118/0.000 ms
```

那么如果是多个ns想联通呢？此时需要先在宿主机上建一个bridge设备，然后把pair全部加进来，那么这些ns就可以联通，然后对这个bridge设置一个Ip，同样做下snat转换，这样，这些ns就都可以访问外网了。而且也实现互联了。

# ns互联

```bash
# 使用刚才上面的ns1环境，现在还需要一个ns
ip netns add ns2

# 创建veth
ip link add name veth00 type veth peer name veth11

ip link set veth11 netns ns2

# 创建bridge
ip lind add name br0 type bridge

# 启动桥设备，并设置ip
ifconfig br0 172.30.3.2/16 up

# 将veth0 veth00放入桥中
ip link set veth0 master br0
ip link set veth00 master br0

# 由于桥中的设备必须是二层的设备，不能有ip，刚才实验环境中设置了veth0的IP,需要把它去掉

ip add del 172.30.1.2/16 dev veth0

# veth1已经有了ip，现在需要把veth11也启动并设置ip
ip netns exec ns2 bash
ip link add name veth00 type veth peer name veth11
ifconfig veth11 172.30.4.2/16 up
ip r add default via 172.30.3.2 dev veth11

# 把veth00 up起来
ifconfig veth00 up
ifconfig veth0 up
```

由于iptables的规则在上面的环境中已经加了，此时直接可以ping通了外网了。

这个实验需要保证：
1. 有bridge, 且bridge 有IP 是up的。  
2. 保证所有的veth peer都是up的
3. 桥中的二层设备不能有ip
4. Iptables要有snat，才能访问外网


# 附

## 相关包

```bash
yum install iproute2
yum install net-tools
```

对于centos，如果没有某个命令，可以直接search下，如：`yum search ifconfig`，无需记包名。

## 参考 

https://www.systutorials.com/docs/linux/man/8-ip-link/

https://hansedong.github.io/2018/12/21/12/

