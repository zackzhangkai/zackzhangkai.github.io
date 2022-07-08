---
layout: post
title:  openstack实战Horizon（7）
date:   2018-05-14  
categories: project
tag:
  - openstack

---
* content
{:toc}

### 安装Horizon
```
[root@linux-node2 ~]# yum install -y openstack-dashboard
```
### Horizon配置
```
[root@linux-node2 ~]# vim /etc/openstack-dashboard/local_settings
OPENSTACK_HOST = "192.168.56.11"
#允许所有主机访问
ALLOWED_HOSTS = ['*', ]
#设置API版本
OPENSTACK_API_VERSIONS = {
    "identity": 3,
    "volume": 2,
    "compute": 2,
}
开启多域支持
OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True
设置默认的域
OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = 'Default'
#设置Keystone地址
OPENSTACK_HOST = "192.168.56.11"
OPENSTACK_KEYSTONE_URL = "http://%s:5000/v3" % OPENSTACK_HOST
#为通过仪表盘创建的用户配置默认的 user 角色
OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"

#设置Session存储到Memcached
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
        'LOCATION': '192.168.56.11:11211',
    }
}
#启用Web界面上修改密码
OPENSTACK_HYPERVISOR_FEATURES = {
    'can_set_mount_point': True,
    'can_set_password': True,
    'requires_keypair': False,
}
#设置时区
TIME_ZONE = "Asia/Shanghai"
#禁用自服务网络的一些高级特性
OPENSTACK_NEUTRON_NETWORK = {
    ...
    'enable_router': False,
    'enable_quotas': False,
    'enable_distributed_router': False,
    'enable_ha_router': False,
    'enable_lb': False,
    'enable_firewall': False,
    'enable_vpn': False,
    'enable_fip_topology_check': False,
}
```

### 启动服务
```
[root@linux-node2 ~]# systemctl enable httpd.service
[root@linux-node2 ~]# systemctl restart httpd.service
```

好的，现在你就可以使用http://192.168.56.12/dashaboard来访问仪表盘了。用户名和密码可以使用admin或者demo。需要你亲自来体验他们到底有什么不同。
