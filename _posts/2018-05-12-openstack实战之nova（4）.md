---
layout: post
title:  openstack实战之nova（4）
date:   2018-05-12
categories: project
tag:
  - openstack

---
* content
{:toc}


### 介绍

目前的Nova主要由API，Compute，Conductor，Scheduler组成

+ Compute：用来交互并管理虚拟机的生命周期；
+ Scheduler：从可用池中根据各种策略选择最合适的计算节点来创建新的虚拟机；
+ Conductor：为数据库的访问提供统一的接口层。

Nova用于为单个用户或使用群组管理虚拟机实例的整个生命周期，根据用户需求来提供虚拟服务。负责虚拟机创建、开机、关机、挂起、暂停、调整、迁移、重启、销毁等操作，配置CPU、内存等信息规格。</p>
1、Nova API ：HTTP服务，用于接收和处理客户端发送的HTTP请求</p>
2、Nova Cert ：用于管理证书，为了兼容AWS。AWS提供一整套的基础设施和应用程序服务，使得几乎所有的应用程序在云上运行</p>
3、Nova Compute ：Nova组件中最核心的服务，实现虚拟机管理的功能。实现了在计算节点上创建、启动、暂停、关闭和删除虚拟机、虚拟机在不同的计算节点间迁移、虚拟机安全控制、管理虚拟机磁盘镜像以及快照等功能。</p>
4、Nova Conductor ：RPC服务，主要提供数据库查询功能。以前的openstack版本中，Nova Compute子服务中定义了许多的数据库查询方法。但是，由于Nova Compute子服务需要在每个计算节点上启动，一旦某个计算节点被攻击，就将完全获得数据库的访问权限。有了Nova Conductor子服务之后，便可在其中实现数据库访问权限的控制。</p>
5、Nova Scheduler ：Nova调度子服务。当客户端向Nova 服务器发起创建虚拟机请求时，决定将续集你创建在哪个节点上。</p>
6、Nova Console、Nova Consoleauth、Nova VNCProxy ：Nova控制台子服务。功能是实现客户端通过代理服务器远程访问虚拟机实例的控制界面。

Compute Service Nova 是 OpenStack 最核心的服务，负责维护和管理云环境的计算资源。
OpenStack 作为 IaaS 的云操作系统，虚拟机生命周期管理也就是通过 Nova 来实现的

Glance 为 VM 提供 image <br>
Cinder 和 Swift 分别为 VM 提供块存储和对象存储 <br>
Neutron 为 VM 提供网络连接<br>

#### Nova下的主要资源
(1) Availability Zone  一组节点的集合
(2) Host-aggregate 按照硬件资源某个属性划分的集合
(3) Instance instance实例相关的东西，包括虚机的生命周期管理、迁移等
(4) Flavor 计算实例的模版，定义计算实例的vCPU、RAM、硬盘大小等
(5) Server-group 虚拟机组，可以指定亲和性/反亲和性等
(6) Quota 控制用户的资源限额


[参考](https://www.cnblogs.com/CloudMan6/p/5716947.html)

### 控制节点安装
```
[root@linux-node1 ~]# yum install -y openstack-nova-api openstack-nova-placement-api \
  openstack-nova-conductor openstack-nova-console \
  openstack-nova-novncproxy openstack-nova-scheduler
```

### 数据库配置
```
[root@linux-node1 ~]# vim /etc/nova/nova.conf
[api_database]
connection= mysql+pymysql://nova:nova@192.168.56.11/nova_api
[database]
connection= mysql+pymysql://nova:nova@192.168.56.11/nova
```

### RabbitMQ配置
```
[root@linux-node1 ~]# vim /etc/nova/nova.conf
[DEFAULT]
transport_url = rabbit://openstack:openstack@192.168.56.11
```

### Keystone相关配置
```
[root@linux-node1 ~]# vim /etc/nova/nova.conf
[api]
auth_strategy=keystone
[keystone_authtoken]
auth_uri = http://192.168.56.11:5000
auth_url = http://192.168.56.11:35357
memcached_servers = 192.168.56.11:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = nova
password = nova
```

### 关闭Nova的防火墙功能

```
[DEFAULT]
use_neutron=true
firewall_driver = nova.virt.firewall.NoopFirewallDriver
```

### VNC配置
```
[root@linux-node1 ~]# vim /etc/nova/nova.conf
[vnc]
enabled=true
server_listen = 0.0.0.0
server_proxyclient_address = 192.168.56.11
```

### 设置glance
```
[glance]
api_servers = http://192.168.56.11:9292
```

### 在 [oslo_concurrency] 部分，配置锁路径：
```
[oslo_concurrency]
lock_path=/var/lib/nova/tmp
```

### 设置启用的api
```
[DEFAULT]
enabled_apis=osapi_compute,metadata
```

### 设置placement
```
[placement]
os_region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://192.168.56.11:35357/v3
username = placement
password = placement
```

### 修改nova-placement-api.conf

```
[root@linux-node1 ~]# vim /etc/httpd/conf.d/00-nova-placement-api.conf
<Directory /usr/bin>
   <IfVersion >= 2.4>
      Require all granted
   </IfVersion>
   <IfVersion < 2.4>
      Order allow,deny
      Allow from all
   </IfVersion>
</Directory>
</VirtualHost>
# systemctl restart httpd
```

### 同步数据库
```
[root@linux-node1 ~]# su -s /bin/sh -c "nova-manage api_db sync" nova
```

### 注册cell0数据库
```
[root@linux-node1 ~]# su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova
```

### 创建cell1的cell
```
[root@linux-node1 ~]# su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova
```

### 同步nova数据库
```
[root@linux-node1 ~]# su -s /bin/sh -c "nova-manage db sync" nova
```

### 验证cell0和cell1的注册是否正确
```
[root@linux-node1 ~]# nova-manage cell_v2 list_cells
```
### 测试数据库同步情况
```
[root@linux-node1 ~]#mysql -h 192.168.56.11 -unova -pnova -e " use nova;show tables;"
[root@linux-node1 ~]#mysql -h 192.168.56.11 -unova -pnova -e " use nova_api;show tables;"
```

### 启动Nova Service
```
# systemctl enable openstack-nova-api.service \
openstack-nova-consoleauth.service \
  openstack-nova-scheduler.service \
openstack-nova-conductor.service \
  openstack-nova-novncproxy.service

# systemctl start openstack-nova-api.service \
  openstack-nova-consoleauth.service \
  openstack-nova-scheduler.service openstack-nova-conductor.service \
  openstack-nova-novncproxy.service
```

### Nova服务注册
```
# source admin-openstack.sh
# openstack service create --name nova --description "OpenStack Compute" compute
# openstack endpoint create --region RegionOne compute public http://192.168.56.11:8774/v2.1
# openstack endpoint create --region RegionOne compute internal http://192.168.56.11:8774/v2.1
# openstack endpoint create --region RegionOne compute admin http://192.168.56.11:8774/v2.1

# openstack service create --name placement --description "Placement API" placement
# openstack endpoint create --region RegionOne placement public http://192.168.56.11:8778
# openstack endpoint create --region RegionOne placement internal http://192.168.56.11:8778
# openstack endpoint create --region RegionOne placement admin http://192.168.56.11:8778
```

### 验证控制节点服务
```
[root@linux-node1 ~]# openstack host list
```

### 计算节点安装
```
[root@linux-node2 ~]# yum install -y openstack-nova-compute sysfsutils

[root@linux-node1 ~]# scp /etc/nova/nova.conf 192.168.56.12:/etc/nova/nova.conf
[root@linux-node2 ~]# chown root:nova /etc/nova/nova.conf
```

### 删除多余的数据配置
### 修改VNC配置
计算节点需要监听所有IP，同时设置novncproxy的访问地址
```
[vnc]
enabled=true
server_listen = 0.0.0.0
server_proxyclient_address = 192.168.56.12
novncproxy_base_url = http://192.168.56.11:6080/vnc_auto.html
```
### 虚拟化适配
```
[root@linux-node2 ~]# egrep -c '(vmx|svm)' /proc/cpuinfo
[libvirt]
virt_type=qemu
```
如果返回的是非0的值，那么表示计算节点服务器支持硬件虚拟化，需要在nova.conf里面设置
```
[libvirt]
virt_type=kvm
```
### 启动nova-compute
```
# systemctl enable libvirtd.service openstack-nova-compute.service
# systemctl start libvirtd.service openstack-nova-compute.service
```
### 验证计算节点
```
[root@linux-node1 ~]# openstack host list
```
### 计算节点加入控制节点
```
[root@linux-node1 ~]# su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova
```
