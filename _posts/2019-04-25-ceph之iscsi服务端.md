---
layout: post
published: true
title:  ceph-iscsi之服务端
categories: [document]
tags: [iscsi,ceph]
---
* content
{:toc}

### 前言
[GITHUB](https://github.com/ceph/ceph-iscsi)

进程  

+ rbd-target-gw daemon
+ gwcli 命令：利用rbd-target-api服务守进程来配置多个网关

安装
```
yum install -y python-rados python-rbd python-netifaces python-rtslib python-configshell python-cryptography python-flask
```

`rpm -ivh ceph-iscsi-<ver>.el7.noarch.rpm`
