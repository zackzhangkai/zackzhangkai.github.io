---
layout:       post
title:        "salt-practice"
date:         2017-03-02 12:00:00
categories: document
tag: salt
---

* content
{:toc}

### 概述
依赖zeromq，salt-master监听两个端口：4505/tcp，publist_port，提供远程执行命令发送功能；4506/tcp，ret_port，用于文件服务、认证、结果搜索等功能接口，master端口需开通4505、4506,minion端口无特定端口要开通，minion端通过订阅master端4505端口的消息，然后将消息通过master端的4506端口返回结果
```
yum install https://repo.saltstack.com/yum/redhat/salt-repo-latest-1.el7.noarch.rpm -y
yum clean expire-cache
yum install salt-master
yum install salt-minion
salt-master -l debug #查看debug信息
salt-minion -l debug #查看minion信息
salt-key -L #查看key
salt-key -A -y #签售证书
rpm -ql salt-master #查看安装时安装了哪些文件（yum）
```

>注意：若安装有epel-release,请先卸载，又可能造成依赖安装不成功，salt官方已有审明

### 操作目标参数
Target options:
+ -E, --pcre 正则匹配
+ -L, --list 列表匹配
+ -G，--grain grain匹配
+ --grain-pcre grains加正则匹配
+ -N, --nodegroup 组匹配
+ -R, --range 范围匹配
+ -C, --compound 综合匹配
+ -I, --pillar pillar值匹配
+ -S, --ipcidr minions网段地址匹配

### 配置文件
/etc/salt/master

### 远程执行模块
salt '目标机器' 函数

+ salt 'node2' sys.list_modules #查看远程执行模块
+ salt 'node2' sys.list_functions pkg #查看pkg执行模块中的所有函数
+ salt 'node2' sys.list_state_modules #状态文件属于状态模块，查看所有状态模块
+ salt 'node2' sys.list_state_functions pkg #查看pkg状态模块中的所有函数
#### cmd.run模块  
/usr/lib/python2.7/site-packages/salt/modules/cmdmod.py
+ `salt 'node2' sys.doc cmd.run`   #查看函数说明
+ `salt 'node2' cmd.run "hostname"`   #举例
+ `salt 'node2' sys.list_functions cmd` #列出cmd模块的函数

>原理都是在匹配的目标机器上执行python模块中的函数，可以编写自己的python模块推送到minion上按上述方式执行
#### pkg.install
+ `salt 'node2' pkg.install httpd`

### 状态系统
状态系统通过sls文件描述minion要达到什么状态，底层由saltstack状态模块保证minion处于该状态。
+ `salt 'node2' pkg.install httpd`
+ 采用状态描述为
```
# cat /srv/salt/apache.sls
install_httpd:
  pkg.installed:
    - name: httpd
```
`salt 'node2' state.apply apache`

>两者都达到了在目标机器上部署httpd服务的目的，但是执行远程命令的方式每次都会执行相同的逻辑和指令，而状态文件则是根据描述让minion处于指定状态，当前状态和所需状态不同时才执行相关操作。

### highstate
通过top.sls作为入口，对模块和主机进行管理，用top.sls组织多个状态文件，对模块进行拆分和复用，实现环境的配置管理；file_roots默认只有一个base环境，/srv/salt，top.sls就在环境的根目录下。

https://github.com/kaizamm/blog/blob/master/1.png



#### top.sls
```
# cat /srv/salt/top.sls
base:
  '*':
    - myapp
```
top.sls中引用了myapp，它会按如下顺序引用：如果存在myapp.sls，则引用myapp.sls；如果不存在，则引用myapp目录下的init.sls。我们采用子目录的方式对目录进行规划。

在myapp/init.sls中include与myapp相关的各个状态文件（可以把它拆分成多个，在此处include）
```
#cat /srv/salt/myapp/init.sls
include:
  - .myconf  
```
myconf对应与init.sls同目录的myconf.sls状态文件，在该文件中对minion的/tmp/myconf.txt状态进行描述。
```
conf1:
  file.managed:
    - name: /tmp/myconf.txt
    - source: salt://myapp/files/myconf.txt
    - user: root
    - group: root
    - mode: 644
```

https://github.com/kaizamm/blog/blob/master/state-file-described.png

在myapp/files/myconf.txt中写入任意内容，执行如下命令让minion处于描述的状态。`salt 'node2' state.apply`

参考：
+ https://docs.saltstack.com/en/latest/topics/tutorials/states_pt1.html
+	https://docs.saltstack.com/en/latest/topics/tutorials/starting_states.html

### grains
saltstack中记录minion静态信息的组件（os类型、cpu核数、内存大小、ip地址等），在minion启动时采集汇报给master，因此grains通常是存储的静态、不常变化的数据，存储在minon本地。minion可以操作自己的grains数据（增删改）可通过如下命令查看每项grains数据。
`salt 'node2' grains.items`。minions的grains信息是minions启动时采集汇报给master的。
+ 查grains信息 `salt '*' sys.list_functions grains`
+ 查看文档信息 `salt '*' sys.doc grains`

### pillar
与grains类似，但pillar存储的相对经常变化的数据，存储在master本地。minion只能查看自己的pillar数据。可通过如下命令查看每项pillar数据： `salt '*' pillar.items`

### jinja
python模板引擎，通过与grains和pillar结合定义动态的配置。如，不同的minion通过fqdn可以获取各自不同的主机名
```
#salt '*' grains.item fqdn
node3:
    ----------
    fqdn:
        node3
node2:
    ----------
    fqdn:
        node2
```

接下来我们可以通过grains创建动态配置，让每个目标机器中的/tmp/myconf.txt显示自己的主机名。修改状态文件中的file描述，指定template为jinja
```
#cat dev/myappconf.sls
conf1:
  file.managed
    - name: /tmp/myconf.txt
    - source: salt://dev/myapp/files/myconf.txt
    - user: root
    - group: root
    - mode: 644
    - template: jinja
```
在source对应的文件中通过 {{grains['ITEM']}}的方式引用grains数据
```
#cat dev/myapp/files/myconf.txt
[BASE] /srv/salt/dev
My hostname is {{grains['ITEM']}}
```
```
# ssh node2
# cat /tmp/myconf.txt
[BASE] /srv/salt/dev
My hostname is node2
```
参考：
+ https://docs.saltstack.com/en/latest/topics/tutorials/states_pt3.html

### 多环境配置和管理
```
# file_roots:
  base:
     - /srv/salt/
   dev:
     - /srv/salt/dev/services
     - /srv/salt/dev/states
   prod:
     - /srv/salt/prod/services
     - /srv/salt/prod/states
```
在多环境下，每个环境的根目录下维护各自的top.sls

https://github.com/kaizamm/blog/blob/master/2.png


分别在base、dev、prod环境的myconf.txt中写入不同内容，执行如下命令让不同环境的Minion处于对应的描述状态
```
salt '*' state.apply  # 默认base环境
salt '*' state.apply saltenv=dev # dev环境
salt '*' state.apply saltenv=prod # prod环境
```
参考：
+	https://docs.saltstack.com/en/latest/topics/tutorials/states_pt4.html

### 总结
+ salt-call test.ping 在minion端运行该语句，查看本地Minion是否正常。在minion端可以通过salt-call service.status <service_name> 查看minion端发送给master的信息; 如：tomcat的daemon（/etc/init.d/tomcat）里在 case语法里需加入 status，以保证service tomcat status显示服务当前运行状态。
+ salt state.service 可管理daemon程序，管理非daemon程序可用supervisor
