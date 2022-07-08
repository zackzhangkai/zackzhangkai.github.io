---
layout:       post
title:        "salt-grains-pillars-jinja"
date:         2017-05-17 12:00:00
categories: document  
tag: salt
---

* content
{:toc}


### grains/pillars及模版基础
grains、pillars提供了一种允许在minion中作用用户自定义变量的方案。模版为这些提供了更高级的用法。grains定义在指定的minion上，pillar定义在master上。它们都可以通过静态（statically)或动态(dynamically)的方式进行定义，但是grain常用于提供不常修改的数据，至少是不重启minion就不会变，而pillar更倾向于动态的数据。

### 使用grain来获取minion的特征数据
grain在设计之初用于描述minion的静态要素，执行模块能够使用它来判断应该如何执行。如os_family的grain数据为debian时，用apt-get管理软件包，为Redhat时使用yum来进行yum来管理。salt会自动发现很多grain数据，如os,os_family,saltversion,pythonversion。grain会在minion进程启动时进行加载，并缓存到内存中。

+ salt 'minionid' grains.items  查看有哪些可用的grain数据
+ salt 'minionid' grains.item os_family 想看特定的grain，将对应的名字作为参数传递给grains.items

### 自定义grain
 在以前的版本中直接写到/etc/salt/minion中定义grain数据
```
grains:
  foo: bar
  baz: qux
```
 这种方式仍在用，但不推荐，推荐直接将grain数据写到/etc/salt/grains中，优点： grain独立存储、grain能通过grain执行模块进行修改
` cat /etc/salt/grains `
```
foo: bar
baz: qux
```
与第一种方式的区别就是没有顶级声明

+ 在minion中增加了一个grain
```
salt 'minionid' grains.setval mygrain 'this is the content of mygrain'
```  

grain值支持多种类型，一般为字符串，也可以为列表
```
my_items:
  - item1
  - item2
```
+ salt 'minionid' grains.append my_items item3  对列表添加项目（item）时，可以用grains.append方法，用该方法时，需grain是一个list，形式如上
+ salt 'minionid' grains.delval my_items    删除一个grain (理论上可行，但在测试时提示无法删除grain）

### 使用pillar使变量集中化
大多数场景下，pillar的表现行为和grain一致，但有一个区别，pillar在master上，grain在minion端。默认情况下，pillar存放在/srv/pillar/目录里。由于该区域存放的是用于众多minion信息，因此需要一种target方式来对应minion。正因如此，所以有了sls文件。pillar的top.sls文件在配置和功能上与state的top.sls文件一致。首先声明一个环境。然后是一个target，最后是该target需要使用的sls文件列表。
```
base:
  '*':
    - pillar-test
```
pillar的sls文件相对于state的sls文件简单许多，因为pillar只提供静态数据存储。以键值对的key/value方式进行定义，有时会包含一定的层级。

`cat /srv/pillar/pillar-test.sls`
```
skel_dir: /etc/skel/
role: web
web_content:
  images:
    - jpg
    - png
    - gif
  scripts:
    - css
    - js
```
与state的sls一样，可用include来引用其他sls文件

+ salt 'minionid' pillar.items  看所有的pillar数据

 由于pillar中有些数据敏感，故可在/etc/salt/master中将pillar_opts改为pillar_opts: False,来关闭输出。除了master配置数据外，pillar数据只能被特定的target minion看到，换句话说，没有minion允许访问其他minion的pillar数据，至少在默认情况下是这样的。salt中是允许使用peer系统来执行master命令的，不过peer系统的内容在本处不做讨论。


### jinja模版
salt支持如下模版引擎：jinja/mako/wempy/cheetah/genshi；数据结构：yaml/yamlex/json/msgpack/py/pyobjects/pydsl。默认情况下，state的sls文件会首先使用jinja进行渲染，后再使用YAML渲染。可在sls文件的第一行中包含shabang行来指定渲染器： #!!py ,shabang也可以以管道符号分隔指定多个渲染器，如想用mako和json来替代默认的jinja和YAML，可以进行如下配置： '#!mako|json'，若要改系统默认，则在/etc/salt/master中修改，renderer: yaml_jinja，也可以在minion上使用file.managed state创建文件时指定模版引擎：
```
apache2_conf:
  file:
    - managed
    - name: /etc/apache2/apache2.conf1
    - source: salt://apache2/apache2_conf
    - template: jinja
```

+ 常见grain pillar的引用

变量可以通过闭合的双大括号来引用，一个叫作user的Grain
```
{% raw %}
 The user {{ grains['user'] }} is referred to here.
 The user {{ pillar['user'] }} is referred to here.
 {% endraw %}
```

pillar类似
```
{% raw %}
The user {{ salt['grains.get']('user', 'larry') }} is referred to here
The user {{ salt['pillar.get']('user', 'larry') }} is referred to here
{% endraw %}
```
如果pillar或grain中没有设置user，则使用默认的larry

```
{% raw %}
The user {{ salt['config.get']('user', 'larry') }} is referred to here
{% endraw %}
```
salt会首先搜索minion配置文件中的值，如果没有找到，则会检查grain，如果还没有，则搜索pillar。如果还没有找到，它会搜索master配置。如果全没有找到，它地使用提供的默认值。
```
{% raw %}
  {% set myvar = 'My Value' %}
{% endraw %}
```
若是无法通过config.get获取到的，可以使用set关键字

由于jinja是基于python的，因此 大多数python的数据类型都是可用的，如列表list，字典dictionary
```
{% raw %}
 {% set mylist = ['apple','orange','bananas'] %}
{% endraw %}
```


```
{% raw %}
{% set mydict = {'favorite pie': 'key lime','favorite cake': 'sacchertorte'} %}
{% endraw %
```
+ if语块，jinja提供逻辑处理，用于定义模版使用哪个部分、如何使用。条件判断使用if块。

```
{% if grains['os_family'] == 'Debian'  %}
apache2:
{% if grains['os_family'] == 'RedHat'  %}
httpd:
{% endif %}
  pkg:
    - installed
  service:
    - running
```
+ for 语块
```
{% raw %}
{% set colors = ['blue','pink'] %}
{% for color in colors %}
{% color %}color
{% endfor %}
{% endraw %}
```
