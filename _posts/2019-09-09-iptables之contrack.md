---
layout: post
published: true
title:  iptables之contrack
categories: [网络]
tags: [iptables, netfilter]
---
* content
{:toc}

# netfilter框架

https://opengers.github.io/openstack/openstack-base-netfilter-framework-overview/

[https://www.frozentux.net/iptables-tutorial/iptables-tutorial.html#STATEMACHINE](https://www.frozentux.net/iptables-tutorial/iptables-tutorial.html#STATEMACHINE)

https://zh.wikipedia.org/wiki/Netfilter
利用运作于用户空间的应用软件，如iptables、ebtables和arptables等，来控制Netfilter，系统管理者可以管理通过Linux操作系统的各种网络数据包。现今许多市面上许多的IP分享器或无线网络路由器（Wireless router），多是嵌入式Linux平台，并利用Netfilter的数据包处理能力，提供NAT以及防火墙的功能。Netfilter平台中制定了五个数据包的挂载点（Hook），分别是PRE_ROUTING、INPUT、OUTPUT、FORWARD与POST_ROUTING。

关于 iptables ct target，请参考
https://lwn.net/Articles/371028/

![](/styles/images/netfilter.png)

我们知道iptables的定位是IPv4 packet filter，它只处理IP数据包，而ebtables只工作在链路层Link Layer处理的是以太网帧(比如修改源目mac地址)。图中用有颜色的长方形方框表示iptables或ebtables的表和链，绿色小方框表示network level，即iptables的表和链。蓝色小方框表示bridge level，即ebtables的表和链，由于处理以太网帧相对简单，因此链路层的蓝色小方框相对较少

iptables是带有状态匹配的防火墙，它使用-m state模块从连接跟踪表查找数据包状态。上面我们分析的那条conntrack条目处于SYN_SENT状态，这是内核记录的状态，数据包在内核中可能会有几种不同的状态，但是映射到用户空间iptables，只有5种状态可用：NEW，ESTABLISHED，RELATED，INVALID和UNTRACKED。注意这里说的状态不是tcp/ip协议中tcp连接的各种状态。下面表格说明了这5种状态分别能够匹配什么样的数据包，注意下面两点
