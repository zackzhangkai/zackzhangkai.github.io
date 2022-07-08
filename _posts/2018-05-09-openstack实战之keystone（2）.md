---
layout: post
title:  openstack实战之keystone（2）
date:   2018-05-09
categories: project
tag:
  - openstack

---
* content
{:toc}

### 安装keystone
```
yum install -y openstack-keystone httpd mod_wsgi memcached python-memcached
```
### Memcache
修改配置文件
```
#vim /etc/sysconfig/memcached
PORT="11211"
USER="memcached"
MAXCONN="1024"
CACHESIZE="64"
OPTIONS="-l 192.168.56.11,::1"  #主要修改IP

systemctl start memcached.service && systemctl enable memcached
```

### keystone配置
配置KeyStone数据库
```
[root@linux-node1 ~]# vim /etc/keystone/keystone.conf
[database]
connection = mysql+pymysql://keystone:keystone@192.168.56.11/keystone
```
设置Token和Memcached
```
[token]
provider = fernet
```
同步数据库
```
[root@linux-node1 ~]# su -s /bin/sh -c "keystone-manage db_sync" keystone
[root@linux-node1 ~]# mysql -h 192.168.56.11 -ukeystone -pkeystone -e " use keystone;show tables;"
```
初始化fernet keys
```
[root@linux-node1 ~]# keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
[root@linux-node1 ~]# keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
```
初始化keystone
```
[root@linux-node1 ~]# keystone-manage bootstrap --bootstrap-password admin \
 --bootstrap-admin-url http://192.168.56.11:35357/v3/ \
 --bootstrap-internal-url http://192.168.56.11:35357/v3/ \
 --bootstrap-public-url http://192.168.56.11:5000/v3/ \
 --bootstrap-region-id RegionOne
```
验证Keystone配置
```
[root@linux-node1 ~]# grep "^[a-z]" /etc/keystone/keystone.conf
connection = mysql+pymysql://keystone:keystone@192.168.56.11/keystone
provider = fernet
```
配置文件修改
```
[root@linux-node1 ~]# vim /etc/httpd/conf/httpd.conf
ServerName 192.168.56.11:80
```
创建配置文件
```
[root@linux-node1 ~]# ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/
```
启动keystone，并查看端口。
```
systemctl enable httpd.service  && \
systemctl start httpd.service
```
设置环境变量

```
#exports.sh
export OS_USERNAME=admin  && \
export OS_PASSWORD=admin && \
export OS_PROJECT_NAME=admin && \
export OS_USER_DOMAIN_NAME=Default && \
export OS_PROJECT_DOMAIN_NAME=Default && \
export OS_AUTH_URL=http://192.168.56.11:35357/v3 && \
export OS_IDENTITY_API_VERSION=3 && \
```
创建项目、用户、角色，并在角色中添加对应的项目和用户
```
openstack project create --domain default --description "Demo Project" demo
openstack user create --domain default --password demo demo
openstack role create user
openstack role add --project demo --user demo user
```
创建Service项目
```
openstack project create --domain default --description "Service Project" service
```
创建glance用户
```
openstack user create --domain default --password glance glance
openstack role add --project service --user glance admin
```
创建nova用户
```
openstack user create --domain default --password nova nova
openstack role add --project service --user nova admin
```
创建placement用户
```
openstack user create --domain default --password placement placement
openstack role add --project service --user placement admin
```
创建Neutron用户
```
openstack user create --domain default --password neutron neutron
openstack role add --project service --user neutron admin
```
创建cinder用户  
```
openstack user create --domain default --password cinder cinder
openstack role add --project service --user cinder admin
```
验证Keystone
```
# unset OS_AUTH_URL OS_PASSWORD
# openstack --os-auth-url http://192.168.56.11:35357/v3 \
--os-project-domain-name default --os-user-domain-name default \
--os-project-name admin --os-username admin token issue
Password:
…
# openstack --os-auth-url http://192.168.56.11:5000/v3 \
--os-project-domain-name default --os-user-domain-name default \
--os-project-name demo --os-username demo token issue
Password:
```

```
#vim /root/admin-openstack.sh
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=admin
export OS_AUTH_URL=http://192.168.56.11:35357/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2

#vim /root/demo-openstack.sh
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=demo
export OS_USERNAME=demo
export OS_PASSWORD=demo
export OS_AUTH_URL=http://192.168.56.11:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
```

```
source admin-openstack.sh && openstack token issue
source demo-openstack.sh  && openstack token issue
```
