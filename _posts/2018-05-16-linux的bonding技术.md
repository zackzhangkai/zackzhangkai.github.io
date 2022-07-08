---
layout: post
title:  linux的bonding技术
date:   2018-05-16  
categories: project
tag:
  - linux

---
* content
{:toc}

### bonding技术

[参考1](https://access.redhat.com/documentation/zh-cn/red_hat_enterprise_linux/7/html/networking_guide/ch-configure_network_bonding)
[参考2](https://access.redhat.com/documentation/zh-cn/red_hat_enterprise_linux/7/html/networking_guide/sec-network_bonding_using_the_command_line_interface#sec-Check_if_Bonding_Kernel_Module_is_Installed)
[参考3](https://www.cnblogs.com/huangweimin/articles/6527058.html)
bonding(绑定)是一种linux系统下的网卡绑定技术，可以把服务器上n个物理网卡在系统内部抽象(绑定)成一个逻辑上的网卡，能够提升网络吞吐量、实现网络冗余、负载等功能，有很多优势

bonding的七种工作模式:

bonding技术提供了七种工作模式，在使用的时候需要指定一种，每种有各自的优缺点.

|        模式          |    说明                |
|----------------------|----------------------------------------------------------------|
|balance-rr (mode=0)   |    默认, 有高可用 (容错) 和负载均衡的功能,  需要交换机的配置，每块网卡轮询发包 (流量分发比较均衡).
|active-backup (mode=1)|  只有高可用 (容错) 功能, 不需要交换机配置, 这种模式只有一块网卡工作, 对外只有一个mac地址。缺点是端口利用率比较低
|balance-xor (mode=2)   |  不常用
|broadcast (mode=3)     |   不常用
|802.3ad (mode=4)       |   IEEE 802.3ad 动态链路聚合，需要交换机配置，没用过
|balance-tlb (mode=5)   |   不常用
|balance-alb (mode=6)   |  有高可用 ( 容错 )和负载均衡的功能，不需要交换机配置  (流量分发到每个接口不是特别均衡)

具体的网上有很多资料，了解每种模式的特点根据自己的选择就行, 一般会用到0、1、4、6这几种模式。

### 环境
centos7 <br>
两张网卡，Vmare虚机请关机后，添加一张物理网卡 <br>
检查是否已安装bonding内核模块
```
modprobe --first-time bonding
modinfo bonding
```
关闭NetworkManager服务
```
systemctl stop NetworkManager.service && systemctl disable NetworkManager.service
```

### 创建频道绑定接口
/etc/sysconfig/network-scripts/ 目录中创建名为 ifcfg-bondN 的文件，使用接口号码替换 N，比如 0

可根据要绑定接口类型的配置文件编写该文件的内容，比如以太网接口。最主要的区别是 DEVICE 指令是 bondN（使用接口号码替换 N）和 TYPE=Bond。此外还设置 BONDING_MASTER=yes。

 ifcfg-bond0 接口配置文件示例

 ```
#vi /etc/sysconfig/network-scripts/ifcfg-bond0

DEVICE=bond0  
IPADDR=192.168.56.13         #根据实际需要，填写需要绑定的ip地址掩码网关  
NETMASK=255.255.255.0
GATEWAY=192.168.56.2
ONBOOT=yes  
BOOTPROTO=none  
USERCTL=no  
BONDING_OPTS="mode=1 primary=eth0 updelay=1000 fail_over_mac=1  miimon=100"  #设置网卡的运行模式，此处配置的是mode=1   miimon是用来进行链路监测的。比如:miimon=100，那么系统每100ms监测一次链路连接状态，如果有一条线路不通就转入另一条线路；模式1为主备模式。

#vi /etc/sysconfig/network-scripts/ifcfg-eth0  
DEVICE=eth0  
BOOTPROTO=none  
ONBOOT=yes  
MASTER=bond0  #很重要的配置，指定物理网卡eth0的Master是bond0  
SLAVE=yes     #指定自己是master的slave角色，受bond0控制

#vi /etc/sysconfig/network-scripts/ifcfg-eth1
DEVICE=eth1  
BOOTPROTO=none  
ONBOOT=yes  
MASTER=bond0 #很重要的配置，指定物理网卡eth0的Master是bond0  
SLAVE=yes    #指定自己是master的slave角色，受bond0控制
 ```
### 加载模块，让系统支持bonding
```
#vim /etc/modprobe.conf 增加一行如下配置：
alias bond0 bonding
```
### 测试
```
systemctl restart network
ip addr
cat /proc/net/bonding/bond0
```
