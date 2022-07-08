---
layout: post
title:  wireshark实战
date:   2018-10-17
categories: document
tag:
  - wireshark

---
* content
{:toc}

### 前言

之前也研究过wireshark，也用过，不过总没有深刻用起来，只知道这个工具很强大，能抓出tcp/ip/http层所有数据，之前侥幸也抓出个简单的，不过，现在在用的时候，还是无法快速要达到自己想要的效果，鉴于此，深入的研究学了一下，并抓出了自己想要的东西。

### 使用

打开wireshark后，选择网卡，确认后开始抓包

<img src="{{ '/styles/images/wireshark/wireshark1.png' | prepend: site.baseurl }}" alt="" width="710" />

在chrome上点击操作，做一次登出登入的操作

<img src="{{ '/styles/images/wireshark/wireshark2-1.png' | prepend: site.baseurl }}" alt="" width="910" />


{#然后再来分析包}

<img src="{{ '/styles/images/wireshark/wireshark2-2.png' | prepend: site.baseurl }}" alt="" width="910" />

如这张图，首先能看到tcp的三次握手，关于 TCP的三次握手：
1. client端发送SYN，并产生一个seq,请求建立连接
2. server收到消息后，回应SYN+ACK，ack=seq+1(client的seq)；此时产生随机seq
3. client端收到消息后，应答ACK,其中ack=seq+1（server产生的seq）

再来分析HTTP，如下图可以看到HTTP的头

<img src="{{ '/styles/images/wireshark/wireshark2-3-1.png' | prepend: site.baseurl }}" alt="" width="910" />

分析一个POST请求，可以直接抓出来用户名密码

<img src="{{ '/styles/images/wireshark/wireshark2-4.png' | prepend: site.baseurl }}" alt="" width="910" />

可以直接过滤包，更快的定位到目标，如下：

<img src="{{ '/styles/images/wireshark/wireshark3.png' | prepend: site.baseurl }}" alt="" width="910" />

<img src="{{ '/styles/images/wireshark/wireshark4.png' | prepend: site.baseurl }}" alt="" width="910" />
