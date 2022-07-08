---
layout:       post
title:        "Django-practice-网页上做加减法"
date:         2017-03-27 12:00:00
categories: project
tag:
  - python
  - django
---

* content
{:toc}

视图与网址

### 网页上做加减法
#### /add/?a=4&b=5 GET方法

blog/views.py
```
#coding:utf-8
from django.shortcuts import render
from django.http import HttpResponse

#def index(request):
#  return HttpResponse(u"hello,world!")

def index(request):
  a = request.GET['a']
  b = request.GET['b']
  c = int(a)+int(b)
  return HttpResponse(str(c))
```

mysite/urls.py
```
from django.conf.urls import url
from django.contrib import admin
from blog import views as blog_views # new

urlpatterns = [
    url(r'^$',blog_views.index,name='index'),   # new
    url(r'^admin/',admin.site.urls),
]
```

浏览器访问   http://172.30.33.183:8000/?a=3&b=9

####  add2/2/3/优雅访问

blog/views.py
```
#coding:utf-8
from django.shortcuts import render
from django.http import HttpResponse


def add2(request,a,b):
  c = int(a) + int(b)

```
mysites/urls.py
```
#coding:utf-8
from django.conf.urls import url
from django.contrib import admin
from blog import views as blog_views # new

urlpatterns = [
    url(r'^$',blog_views.index,name='index'),   # 对应blog/views.py里的对应的index函数
    url(r'^add2/(\d+)/(\d+)/$',blog_views.add2,name='add2'),   # 对应blog/views.py里对应的add2函数,(\d+)表示一个或多个数字，用括号表示一个子组,每个子组将作为一个参数，被view.py中的对应视图函数接收。
    url(r'^admin/',admin.site.urls),
]
```
浏览器访问  http://172.30.33.183:8000/add2/2/3/
