---
layout:       post
title:        "salt-modules"
date:         2017-03-15 12:00:00
categories: document
tag: salt
---

* content
{:toc}

更多参考 https://docs.saltstack.com/en/latest/ref/modules/all/salt.modules.status.html
### cp模块
cp模块实现远程文件、目录的复制，以及下载url文件等操作，参数：cache_local、get_file、get_dir、get_url

```
# 将主服务器file_roots指定位置下的目录复制到被控主机
salt '*' cp.get_dir salt://files/ /data

# 将主服务器file_roots指定位置下的文件复制到被控主机
salt '*' cp.get_dir salt://files/test.txt /data

# 下载指定url内容到主机指定位置
salt '*' cp.get_url http://www.xyz.com/download /root/files.tgz

API调用：client.cmd（'*'，'cp.get_dir'，['salt://path/to/dir'，'/minion/dest']）
```

### cmd模块
默认是root权限，实现远程命令调用执行
```
salt '*' cmd.run 'uname -a'

API调用：client.cmd（'*'，'cmd.run'，['命令']）
```
### cron模块
实现被控主机的crontab操作，增：cron.set_job；查：raw_cron; 删：rm_job
```
#为指定的被控主机、root用户添加crontab信息
salt '*' cron.set_job root '*/5' '*' '*' '*' '*' 'date > /dev/null 2>&1'
salt '*' cron.raw_cron root

# 删除指定的被控主机，root用户crontab信息
salt '*' cron.rm_job root 'date >/dev/null 2>&1'

API调用：client.cmd（'*'，'cron.rm_job'，['root'，'命令']
```

### dnsutil模块
实现被控主机的通用dns配置
```
#为被控主机添加指定hosts主机配置项
salt '*' dnsutil.hosts_append /etc/hosts 127.0.0.1 test.com
API调用：client.cmd（'*'，'dnsutil.hosts_remove'，['/etc/hosts'，'域名']
```
### file模块
在系统层面实现压缩包的调用，gzip/gunzip/tar/unzip等
```
salt '*' archive.gzip /tmp/abc.txt
API调用：client.cmd（'*'，'archive.gunzip '，['/tmp/qq.gz']）
```
### network模块
返回被控主机网络信息,参数有 dig/ping/traceroute 跟域名、hwaddr跟网卡、in_subnet 跟网段、interfaces、interfaces、ip_addrs、subnet无字段
```
salt '*' network.ip_addrs
salt '*' network.interfaces
API调用：client.cmd（'*'，'network.in_subnet'，'网段'）
```
### pkg包管理
被控主机程序包管理，yum及apt-get等，参数有install、remove跟软件包，upgrade升级所有
```
salt '*' pkg.install httpd
salt '*' pkg.file_list httpd
```
### service包管理模块
被控主机程序包服务管理，参数有enable/disable/reload/restart/start/stop/status
```
salt '*' service.enable crond
salt '*' service.reload crond
API调用：client.cmd（'*'，'service.stop' 'httpd'）
```
