---
layout: post
published: true
title:  ceph-iscsi之客户端
categories: [document]
tags: [iscsi,ceph]
---
* content
{:toc}

### 前言
[csdn](https://www.cnblogs.com/netonline/p/10432653.html)
[官网](https://access.redhat.com/documentation/en-us/red_hat_ceph_storage/3/html/block_device_guide/using_an_iscsi_gateway#monitoring_the_iscsi_gateways)
[参考一](https://www.cyberciti.biz/tips/rhel-centos-fedora-linux-iscsi-howto.html)
[鸟哥](http://cn.linux.vbird.org/linux_server/0460iscsi.php)


iSCSI（Internet Small Computer System Interface，发音为/ˈаɪskʌzi/），Internet小型计算机系统接口，又称为IP-SAN，是一种基于因特网及SCSI-3协议下的存储技术，由IETF提出，并于2003年2月11日成为正式的标准。与传统的SCSI技术比较起来，iSCSI技术有以下三个革命性的变化：

1. 把原来只用于本机的SCSI协议透过TCP/IP网络发送，使连接距离可作无限的地域延伸；
2. 连接的服务器数量无限（原来的SCSI-3的上限是15）；
3. 由于是服务器架构，因此也可以实现在线扩容以至动态部署。

iSCSI利用了TCP/IP的port 860 和 3260 作为沟通的渠道。透过两部计算机之间利用iSCSI的协议来交换SCSI命令，让计算机可以透过高速的局域网集线来把SAN模拟成为本地的储存设备。

本质上，iSCSI 让两个主机通过 IP 网络相互协商然后交换 SCSI 命令，这样一来，iSCSI 就是用广域网仿真了一个常用的高性能本地存储总线，从而创建了一个存储局域网（SAN）。不像某些 SAN 协议，iSCSI 不需要专用的电缆；它可以在已有的交换和 IP 基础架构上运行

LUN（Logical Unit Number，逻辑单元号）是为了使用和描述更多设备及对象而引进的一个方法，每个SCSI ID上最多有32个LUN，一个LUN对应一个逻辑设备。

iSCSI 主要是透过 TCP/IP 的技术，将储存设备端透过 iSCSI target (iSCSI 目标) 功能，做成可以提供磁盘的服务器端，再透过 iSCSI initiator (iSCSI 初始化用户) 功能，做成能够挂载使用 iSCSI target 的客户端


# iscsi-gateway服务
```
# 服务需要在所有iscsi-gateway节点启动，以ceph01节点为例；
# 在启动”rbd-target-api”服务的同时，会启动”rbd-target-gw”服务；
# 注意提前创建”rbd” pool，rbd-target-api依赖于rbd-target-gw，rbd-target-gw服务依赖于”rbd”池
[root@ceph01 ~]# systemctl daemon-reload
[root@ceph01 ~]# systemctl enable rbd-target-api
[root@ceph01 ~]# systemctl start rbd-target-api
[root@ceph01 ~]# systemctl status rbd-target-api ; systemctl status rbd-target-gw
```

### 安装 gwtop

```
#安装ceph-iscsi-tools，依旧找不到安装包
git clone https://github.com/ceph/ceph-iscsi-tools.git
cd ceph-iscsi-tools
python setup.py install --install-scripts=/usr/bin

#安装pcp和安装启动pcp-pmda-lio
yum install pcp
yum install pcp-pmda-lio
systemctl enable pmcd
systemctl start pmcd

#注册pcp-pmda-lio代理
cd /var/lib/pcp/pmdas/lio
./Install
```

### 架构
iSCSI 这个架构主要将储存装置与使用的主机分为两个部分，分别是：

1. iSCSI target：就是储存设备端，存放磁盘或 RAID 的设备，目前也能够将 Linux 主机仿真成 iSCSI target 了！目的在提供其他主机使用的『磁盘』；
2. iSCSI initiator：就是能够使用 target 的客户端，通常是服务器。 也就是说，想要连接到 iSCSI target 的服务器，也必须要安装 iSCSI initiator 的相关功能后才能够使用 iSCSI target 提供的磁盘就是了。

![](/styles/images/iscsi.gif)

### das san nas
现在重温下das san nas之间的关系
![](/styles/images/das_nas_san.gif)

从上图中，我们可以发现在一般的主机环境下，磁盘装置 (SATA, SAS, FC) 可以透过主机的接口 (DAS) 来直接进行文件系统的建立 (mkfs 进行格式化)，如果想要使用外部的磁盘，那可以透过 SAN (内含多个磁盘的设备)，然后透过 iSCSI 等接口来联机， 当然，还是得要进行格式化等动作 (假设这个 SAN 尚未被使用时)。最后，如果是 NAS 的条件下，那么 NAS 必须要先透过自己的操作系统将磁盘装置进行文件系统的建立后，再以 NFS/CIFS 等方式来提供其他主机挂载使用。

### 所需软件与软件结构tgt
CentOS 将 tgt 的软件名称定义为 scsi-target-utils ，因此你得要使用 yum 去安装他才行。至于用来作为 initiator 的软件则是使用 linux-iscsi 的项目，该项目所提供的软件名称则为 iscsi-initiator-utils 。所以，总的来说，你需要的软件有：

scsi-target-utils：用来将 Linux 系统仿真成为 iSCSI target 的功能；
iscsi-initiator-utils：挂载来自 target 的磁盘到 Linux 本机上

那么 scsi-target-utils 主要提供哪些档案呢？基本上有底下几个比较重要需要注意的：

/etc/tgt/targets.conf：主要配置文件，设定要分享的磁盘格式与哪几颗；
/usr/sbin/tgt-admin：在线查询、删除 target 等功能的设定工具；
/usr/sbin/tgt-setup-lun：建立 target 以及设定分享的磁盘与可使用的客户端等工具软件。
/usr/sbin/tgtadm：手动直接管理的管理员工具 (可使用配置文件取代)；
/usr/sbin/tgtd：主要提供 iSCSI target 服务的主程序；
/usr/sbin/tgtimg：建置预计分享的映像文件装置的工具 (以映像文件仿真磁盘)；

###  target 的实际设定
从上面的分析来看，iSCSI 就是透过一个网络接口，将既有的磁盘给分享出去就是了。那么有哪些类型的磁盘可以分享呢？ 这包括：
```
使用 dd 指令所建立的大型档案可供仿真为磁盘 (无须预先格式化)；
使用单一分割槽 (partition) 分享为磁盘；
使用单一完整的磁盘 (无须预先分割)；
使用磁盘阵列分享 (其实与单一磁盘相同方式)；
使用软件磁盘阵列 (software raid) 分享成单一磁盘；
使用 LVM 的 LV 装置分享为磁盘。
```

### 规划分享的 iSCSI target 檔名
iSCSI 有一套自己分享 target 档名的定义，基本上，藉由 iSCSI 分享出来的 target 檔名都是以 iqn 为开头，意思是：『iSCSI Qualified Name (iSCSI 合格名称)』的意思。那么在 iqn 后面要接啥档名呢？通常是这样的：
```
iqn.yyyy-mm.<reversed domain name>:identifier
iqn.年年-月.单位网域名的反转写法  :这个分享的target名称
鸟哥做这个测试的时间是 2011 年 8 月份，然后鸟哥的机器是 www.centos.vbird ，反转网域写法为 vbird.centos， 然后，鸟哥想要的 iSCSI target 名称是 vbirddisk ，那么就可以这样写：

iqn.2011-08.vbird.centos:vbirddisk
```
另外，就如同一般外接式储存装置 (target 名称) 可以具有多个磁盘一样，我们的 target 也能够拥有数个磁盘装置的。 每个在同一个 target 上头的磁盘我们可以将它定义为逻辑单位编号 (Logical Unit Number, LUN)。我们的 iSCSI initiator 就是跟 target 协调后才取得 LUN 的存取权就是了。在鸟哥的这个简单案例中，最终的结果，我们会有一个 target ，在这个 target 当中可以使用三个 LUN 的磁盘。

### 安装
```
yum install iscsi-initiator-utils
```

### iscsi配置
要使用iscsi存储需要三步


1. 配置iscsid.conf

```
# vi /etc/iscsi/iscsid.conf
```
设置用户名密码
```
node.session.auth.username = My_ISCSI_USR_NAME
node.session.auth.password = MyPassword
discovery.sendtargets.auth.username = My_ISCSI_USR_NAME
discovery.sendtargets.auth.password = MyPassword
```

2. 发现目标 Discover targets
```
# iscsiadm -m discovery -t st -p 192.168.1.5
# ll -R /var/lib/iscsi/nodes/
# /etc/init.d/iscsi restart
显示出目前系统上面所有的 target 数据,-m node：找出目前本机上面所有侦测到的 target 信息，可能并未登入喔
# iscsiadm -m node
仅登入某部 target ，不要重新启动 iscsi 服务
#  iscsiadm -m node -T target名称 --login
```
这样你在/dev下应该可以发现盘
```
lsblk
fdisk -l
tail -f /var/log/messages
```
/dev/sdd 是新块设备

3. 格式化并挂载iscsi卷

```
# fdisk /dev/sdd
# mke2fs -j -m 0 -O dir_index /dev/sdd1
```
或者
```
# mkfs.ext3 /dev/sdd1
```
若超过1T，可以在后台运行
```
# nohup mkfs.ext3 /dev/sdd1 &
```
挂载新分区
```
# mkdir /mnt/iscsi
# mount /dev/sdd1 /mnt/iscsi
```

4. 开机自动挂载
```
# chkconfig iscsi on
```

### 更新/删除/新增 target 数据的方法

如果你的 iSCSI target 可能因为某些原因被拿走了，或者是已经不存在于你的区网中，或者是要送修了～ 这个时候你的 iSCSI initiator 总是得要关闭吧！但是，又不能全部关掉 (/etc/init.d/iscsi stop)， 因为还有其他的 iSCSI target 在使用。这个时候该如何取消不要的 target 呢？很简单！流程如下
```
# iscsiadm -m node -T targetname --logout  #注意target类似于iqn.2011-08.vbird.centos:vbirddisk
iscsiadm -m node -o [delete|new|update] -T targetname

范例：关闭来自鸟哥的 iSCSI target 的数据，并且移除链接
[root@clientlinux ~]# iscsiadm -m node   <==还是先秀出相关的 target iqn 名称
192.168.100.254:3260,1 iqn.2011-08.vbird.centos:vbirddisk
[root@clientlinux ~]# iscsiadm -m node -T iqn.2011-08.vbird.centos:vbirddisk \
>  --logout
Logging out of session [sid: 1, target: iqn.2011-08.vbird.centos:vbirddisk,
 portal: 192.168.100.254,3260]
Logout of [sid: 1, target: iqn.2011-08.vbird.centos:vbirddisk, portal:
 192.168.100.254,3260] successful.
# 这个时候的 target 连结还是存在的，虽然注销你还是看的到！

[root@clientlinux ~]# iscsiadm -m node -o delete \
>  -T iqn.2011-08.vbird.centos:vbirddisk
[root@clientlinux ~]# iscsiadm -m node
iscsiadm: no records found! <==嘿嘿！不存在这个 target 了～

[root@clientlinux ~]# /etc/init.d/iscsi restart
```
简单介绍
```
iscsiadm的使用分为三步
第一：发现目标（发现成功后会在/var/lib/iscsi/nodes/目录下生成响应的记录文件）
	iscsiadm --mode discovery --type sendtargets --portal 192.168.1.55
	iscsiadm -m discovery -t sendtargets -p 192.168.1.1:3260
第二：登录节点
	iscsiadm -m node –T iqn.1994-05.com.redhat:wsfnk -p 192.168.1.1：3260 -l
	iscsiadm -m node –T iqn.1994-05.com.redhat:wsfnk -p 192.168.1.1:3260 -o update -n node.startup -v automatic	#在系统启动时自动登录
	iscsiadm -m node -T iqn.1994-05.com.redhat:wsfnk -l	#（登陆某个目标器）
	iscsiadm -m node -L all				#（登陆发现的所有目标器）
	iscsiadm -d2 -m node --login			#（登陆发现的所有目标器）
第三：登出节点
	iscsiadm -m node -T iqn.1994-05.com.redhat:wsfnk -u	#（退出某个目标器）
	iscsiadm -m node -U all		#（退出所有登陆的目标器）
	iscsiadm -d2 -m node --logout	#（退出所有登陆的目标器）
	iscsiadm -m node -o delete –T  iqn.1994-05.com.redhat:wsfnk -p 192.168.14.112		#连接死掉（断网或者target端断掉）时
```

### 文件的介绍
```
/etc/iscsi/iscsid.conf：主要的配置文件，用来连结到 iSCSI target 的设定；
/sbin/iscsid：启动 iSCSI initiator 的主要服务程序；
/sbin/iscsiadm：用来管理 iSCSI initiator 的主要设定程序；
/etc/init.d/iscsid：让本机模拟成为 iSCSI initiater 的主要服务；
/etc/init.d/iscsi：在本机成为 iSCSI initiator 之后，启动此脚本，让我们可以登入 iSCSI target。所以 iscsid 先启动后，才能启动这个服务。为了防呆，所以 /etc/init.d/iscsi 已经写了一个启动指令， 启动 iscsi 前尚未启动 iscsid ，则会先呼叫 iscsid 才继续处理 iscsi 喔！
```
