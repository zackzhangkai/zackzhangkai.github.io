---
layout: post
title:  openstack实战之Cinder（8）
date:   2018-05-14  
categories: project
tag:
  - openstack

---
* content
{:toc}


### 控制节点部署

### Cinder安装
```
[root@linux-node1 ~]# yum install -y openstack-cinder
```
### 数据库配置
```
[root@linux-node1 ~]# vim /etc/cinder/cinder.conf
#在 [database] 部分，配置数据库访问。
connection=mysql+pymysql://cinder:cinder@192.168.56.11/cinder
同步数据库
[root@linux-node1 ~]# su -s /bin/sh -c "cinder-manage db sync" cinder
验证数据库状态
[root@linux-node1 ~]# mysql -h 192.168.56.11 -ucinder -pcinder -e "use cinder;show tables;"
```
### Keystone相关配置
```
[root@linux-node1 ~]# vim /etc/cinder/cinder.conf
[DEFAULT]
auth_strategy=keystone
[keystone_authtoken]
auth_uri = http://192.168.56.11:5000
auth_url = http://192.168.56.11:35357
memcached_servers = 192.168.56.11:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = cinder
password = cinder
```
### RabbitMQ相关配置
```
[root@linux-node1 ~]# vim /etc/cinder/cinder.conf
[DEFAULT]
transport_url = rabbit://openstack:openstack@192.168.56.11
```
### 其它配置
```
在 [oslo_concurrency] 部分，配置锁路径：
[oslo_concurrency]
lock_path = /var/lib/cinder/tmp
```
### 配置Nova以使用块设备存储，注意所有
编辑文件 /etc/nova/nova.conf 并添加如下到其中：
```
[cinder]
os_region_name = RegionOne
```
### 重启nova-api服务
```
[root@linux-node1 ~]# systemctl restart openstack-nova-api.service
```
### 启动cinder服务，并设置为开机自动启动
```
# systemctl enable openstack-cinder-api.service openstack-cinder-scheduler.service
# systemctl start openstack-cinder-api.service openstack-cinder-scheduler.service
```
### Cinder注册Service和Endpoint
```
# openstack service create --name cinderv2 --description "OpenStack Block Storage" volumev2
# openstack service create --name cinderv3 --description "OpenStack Block Storage" volumev3
# openstack endpoint create --region RegionOne \
  volumev2 public http://192.168.56.11:8776/v2/%\(project_id\)s
# openstack endpoint create --region RegionOne \
  volumev2 internal http://192.168.56.11:8776/v2/%\(project_id\)s
# openstack endpoint create --region RegionOne \
  volumev2 admin http://192.168.56.11:8776/v2/%\(project_id\)s


# openstack endpoint create --region RegionOne \
  volumev3 public http://192.168.56.11:8776/v3/%\(project_id\)s
# openstack endpoint create --region RegionOne \
  volumev3 internal http://192.168.56.11:8776/v3/%\(project_id\)s
 # openstack endpoint create --region RegionOne \
  volumev3 admin http://192.168.56.11:8776/v3/%\(project_id\)s
```
### 存储节点配置
对于CentOS环境，默认是已经安装了LVM。如果没有可以使用以下命令安装并启动。
### 安装 LVM 包：
```
[root@linux-node1 ~]# yum install -y lvm2 device-mapper-persistent-data
```
启动LVM的metadata服务并且设置该服务随系统启动：
```
[root@linux-node1 ~]# systemctl enable lvm2-lvmetad.service
[root@linux-node1 ~]# systemctl start lvm2-lvmetad.service
```
把/dev/sdb创建为LVM的物理卷：
```
[root@linux-node2 ~]# pvcreate /dev/sdb
  Physical volume "/dev/sdb" successfully created
```

创建名为cinder-volumes的逻辑卷组
```
[root@linux-node2 ~]# vgcreate cinder-volumes /dev/sdb
  Volume group "cinder-volumes" successfully created
[root@linux-node2 ~]# vim /etc/lvm/lvm.conf
    在``devices``部分，添加一个过滤器，只接受``/dev/sdb``设备，拒绝其他所有设备：
    devices {
    ...
    filter = [ "a/sdb/", "r/.*/"]
    filter = [ "a/sda/", "a/sdb/", "r/.*/"]
    filter = [ "a/sda/", "r/.*/"]
```

### 存储节点安装

   存储节点安装和控制节点类型，还是分为两步：

1.    软件安装。
2.    从控制节点SCP配置文件。

### 安装isci-target和cinder
```
[root@linux-node2 ~]# yum install -y openstack-cinder targetcli python-keystone
```
### 同步控制节点配置文件

由于存储节点大多数配置和控制节点相同，可以直接使用控制节点配置好的cinder.conf。再此基础上进行小的变动。
```
[root@linux-node1 ~]# scp /etc/cinder/cinder.conf 192.168.56.12:/etc/cinder/
```
### 设置Cinder后端驱动
```
[root@linux-node2 ~]# vim /etc/cinder/cinder.conf
[lvm]
volume_driver = cinder.volume.drivers.lvm.LVMVolumeDriver
volume_group = cinder-volumes
iscsi_protocol = iscsi
iscsi_helper = lioadm
volume_backend_name=iSCSI-Storage

在 [DEFAULT] 部分，启用 LVM 后端：
[DEFAULT]
...
enabled_backends = lvm


[DEFAULT]
glance_api_servers = http://192.168.56.11:9292
```

启动块存储卷服务及其依赖的服务，并将其配置为随系统启动：
 # systemctl enable openstack-cinder-volume.service target.service
 # systemctl start openstack-cinder-volume.service target.service
