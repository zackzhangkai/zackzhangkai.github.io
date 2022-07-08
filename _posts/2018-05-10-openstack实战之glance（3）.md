---
layout: post
title:  openstack实战之glance（3）
date:   2018-05-10
categories: project
tag:
  - openstack

---
* content
{:toc}

### 安装glance
```
yum install -y openstack-glance
```
### 数据库配置
Glance-api.conf
```
#vim /etc/glance/glance-api.conf
[database]
connection= mysql+pymysql://glance:glance@192.168.56.11/glance
```
glance-registry.conf
```
#vim /etc/glance/glance-registry.conf
[database]
connection= mysql+pymysql://glance:glance@192.168.56.11/glance
```

### 设置Keystone  
```
#vim /etc/glance/glance-api.conf
[keystone_authtoken]
auth_uri = http://192.168.56.11:5000
auth_url = http://192.168.56.11:35357
memcached_servers = 192.168.56.11:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = glance
password = glance

[paste_deploy]
flavor=keystone
```
glance-registry.conf配置
```
#vim /etc/glance/glance-registry.conf
auth_uri = http://192.168.56.11:5000
auth_url = http://192.168.56.11:35357
memcached_servers = 192.168.56.11:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = glance
password = glance

[paste_deploy]
flavor=keystone
```

### 设置Glance镜像存储
```
#vim /etc/glance/glance-api.conf
[glance_store]
stores = file,http
default_store=file
filesystem_store_datadir=/var/lib/glance/images/
```

### 同步数据库
```
su -s /bin/sh -c "glance-manage db_sync" glance
```
### 启动glance服务

```
systemctl enable openstack-glance-api.service && systemctl start openstack-glance-api.service
systemctl enable openstack-glance-registry.service && systemctl start openstack-glance-registry.service
```
### Glance服务注册
想要让别的服务可以使用Glance，就需要在Keystone上完成服务的注册。注意需要先source一下admin的环境变量。
```
source admin-openstack.sh
openstack service create --name glance --description "OpenStack Image service" image
openstack endpoint create --region RegionOne   image public http://192.168.56.11:9292
openstack endpoint create --region RegionOne   image internal http://192.168.56.11:9292
openstack endpoint create --region RegionOne   image admin http://192.168.56.11:9292
```
### 测试Glance状态
```
source admin-openstack.sh
openstack image list
```

### Glance镜像
在刚开始实施OpenStack平台阶段，如果没有制作镜像。可以使用一个实验的镜像进行测试，这是一个小的Linux系统。

```
cd /usr/local/src &&  wget http://download.cirros-cloud.net/0.3.5/cirros-0.3.5-x86_64-disk.img
openstack image create "cirros" --disk-format qcow2 --container-format bare --file cirros-0.3.5-x86_64-disk.img --public
openstack image list
```
