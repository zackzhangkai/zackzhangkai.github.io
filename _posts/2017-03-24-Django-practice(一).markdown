---
layout:       post
title:        "Django-practice(一)"
date:         2017-03-24 12:00:00
categories: document
tag:
  - django
  - python
---

* content
{:toc}

### 安装
```
##pip安装 python2.7
# 方法一：yum install python-pip
# 方法二：源码编译安装
wget https://pypi.python.org/packages/source/p/pip/pip-1.5.4.tar.gz
tar zxf pip-1.5.4.tar.gz && cd pip-1.5.4 && python setup.py install
#django安装
pip install Django
#检查是否安装成功
import django
help(django) #能出现django的version
```

### Practice
django 1.10.6
#### 新建project
`django-admin startproject mysite`

会生成以下文件

├── manage.py
└── mysite
    ├── __init__.py # python包的目录结构必须的，与调用有关
    ├── settings.py  # 目录 mysite 中是一些项目的设置，配置文件；比如DEBUG的开关，静态文件的位置等。
    ├── urls.py   # 总的urls配置文件，网址入口；关联到对应的views.py中的一个函数（或者generic类），访问网址就对应一个函数。
    └── wsgi.py  # 部署服务器时用到的



#### 新建app
一个项目中可以有多个app,当然通用的app可以在多个项目中使用

`python manage.py startapp blog`  

会形成生成以下文件

├── admin.py  # 后台，可以用很少量的代码就拥有一个强大的后台
├── apps.py #
├── __init__.py
├── migrations
│   └── __init__.py
├── models.py # 与数据库操作相关，存入或读取数据时用到这个，当然用不到数据库的时候可以不用
├── tests.py
└── views.py # 处理用户发出的请求，从urls.py中对应过来，通过渲染templates中的网页可以将显示内容，比如登陆后的用户名，用户的请求的数据，输出到网页。views.py中的函数渲染templates中的html模版，得到动态内容的网，可以用来缓存，提高速度

+ 把新定义的app加入到settings.py中的INSTALL_APPS中,django就能自动找到app中的模板文件(blog/templates/下的文件)和静态文件(blog/static/中的文件)
```
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',

    'blog',
]
...
ALLOWED_HOSTS = ["172.30.33.183",]  # 在写上HOST IP
...
```

+ 定义视图函数（访问页面时的内容）
blog目录下的views.py改成下面的代码：
```
#coding:utf-8   
from django.shortcuts import render
from django.http import HttpResponse

def index(request):
  return HttpResponse(u"hello,world!")
```
第一行声明编码utf-8，后面可以用中文；第三行引入HttpResponse，它是用来向网页返回内容的，就像python中的print一样，只不过HttpResponse是把内容显示到网页上；后面我们定义了一个inidex()函数，第一个参数必须是request，与网页发来的请求有关，request变量里面包含了get或post的内容，用户浏览器，系统等信息在里面；函数返回一个HttpResponse对象，可以经过一些处理，最终显示几个字到网页上。关于网址在mysite/mysite/urls.py中规定会网址对应什么内容。打开urls.py修改其中的代码：
```
from django.conf.urls import url
from django.contrib import admin
from blog import views as blog_views # new

urlpatterns = [
    url(r'^$',blog_views.index),   # new
    url(r'^admin/',admin.site.urls),
]
~
```
+ 运行 `python manage.py runserver 0.0.0.0:8000`

网页即可查看对应服务器上的 hello,world!

#### 同步数据库
`python manage.py migrate`

这种方法可以创建表，当你在models.py中新增了类时，运行它就可以自动在数据库中创建表了，不用手动创建。

#### 运行
`python manage.py runserver 0.0.0.0:8000` # 所有机器均可访问对就服务器的http://IP:8000

#### 清空数据库
`python manage.py flush`

#### 创建超级管理员
+ `python manage.py createsuperuser`  创建用户
+ `python manage.py changepassword username` 修改用户的密码

#### 导入、导出数据
python manage.py dumpdata blog > blog.json
python manage.py loaddata blog blog.json

#### django项目环境终端
+ `python manage.py shell`  建议安装ipython 或 bpython,会自动使用该界面，这个命令进入shell后，你可以在这个shell里面调用当前项目的models.py中的API,对于操作数据，还有一些小测试非常方便。
+ 数据库命令行：python manage.py dbshell，Django会自动进入在settings.py中设置的数据库，如果是mysql或postgresql，会要求输入数据库用户密码；在这个终端可以执行sql语句。
参考  http://www.ziqiangxuetang.com/django/django-basic.html  或者直接看官网 https://docs.djangoproject.com/en/1.10/ref/django-admin/

### 视图与网址
