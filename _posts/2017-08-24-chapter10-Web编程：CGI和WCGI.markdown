---
layout:       post
title:        "chapter10-Web编程：CGI和WCGI"
date:         2017-08-24 12:00:00
categories: document
tag: python
---

* content
{:toc}

### CGI
CGI在Web服务器和应用之间充当了交互作用，这样才能够处理用户表单，生成并返回最终的动态HTML页。含有需要用户输入项（如文本框、单选按钮等）、submit按钮、图片的Web页面，都会涉及某种CGI活动。创建HTML的CGI应用程序通常是用高级编程语言来实现的，可以接受、处理用户数据，向服务器返回HTML页面。在接触CGI之前，需要告诫的是，一般生产环境不再使用CGI了，因为其限制web服务器同时处理客户端的数量。如今web服务器典型的部件有Apache和集成的数据库访问部件（Mysql或者PostgreSQL）、Java(Tomcat)、PHP和各种动态语言（python或Ruby）模块，以及SSL/security。然而，如果在小型的私人web网站或者小组织的web网站上工作，就没有必要使用这些用于关键任务的强大需而复杂的web服务器。在开发小型网站或为了测试时，可以使用CGI。虽然Web应用程序开发框架和内容管理系统，这些工具淘汰了CGI，然而，这些新工具虽然进行了抽象和浓缩，但是仍然使用CGI最初提供的模式，如获取用户输入的信息，根据输入执行相关代码，并提供一个有效的HTML作为最终输出传递给客户端
### CGI应用程序
CGI应用程序和典型的应用程序有些不同，主要的区别在于输入、输出以及用户和程序交互方面。当一个CGI脚本启动后，需要获得用户提供的表单数据，但这些数据必须要从Web客户端才可获得，而不是从服务器或者硬盘上获得。这就是大家熟知的Request请求。与标准输出不同，这些输出将会发送回连接的Web客户端，而不是发送到屏幕、CGI窗口或者硬盘上。这些返回的数据必须具有一系列的有效头文件的HTML标签数据。如果web客户端是浏览器，由于浏览器只能识别有效的HTTP数据(也就是MIME头和HTML)所以会发生错误(具体一点就是内部服务器错误)，所以最后需要说明的是：用户和脚本没有任何的交互，所有的交互都将发生在web客户端(基于用户行为)、web服务器端和CGI应用程序间。
### cgi模块
cgi模块有个主要类： FieldStorage类，其完成了所有的工作。Python脚本启动时会实例化这个类，通过Web服务器从Web客户端读出相关的用户信息。在实例化完成后，基中会有一个类似字典的对象，它具有一系列的键值对。键就是通过通过表单传入的表单条目的名字，而值则包含相应的数据。这个值可以是以下三个对象之一。一是FieldStorage对象（实例）。二是另一个名为MiniFieldStorage类的类似实例，用在没有文件上传或multiple-part格式数据的情况下。MiniFieldStorage实例只包含名程和数据的键值对。最后，它们还可以是这些对象的列表 。当表单中的某个字段有多个输入值就会产生这种对象。对于简单的web表单，可以发现其中所有的MiniFieldStorage实例。
### cgitb模块
前面已经提到，返回Web服务器的合法响应（将会转发给用户浏览器）必须含有合法的HTTP头和THML标记过的数据。是否考虑过在CGI应用崩溃时如何返回数据呢。想一想如果是一个python脚本发生错误呢？对，会出现回溯消息。那么回溯的文本消息是否会被认为是合法的HTML头或HTML？不会。web服务器在收到无法理解的响应时，会抛弃这个响应，返回“500错误”。500是一个响应编码，它表示内部服务器错误。一般是应用程序发生了错误。python程序在命令行或IDE中运行时，发生的错误会生成回溯消息，指出错误发生的位置，在浏览器中不会显示回溯消息。若想在浏览器中看到的是web应用程序的回溯，指出错误发生的位置，则可以使用cgitb模块。为了启用转储回溯消息，所要做的就是将下面的代码插入CGI应用中并进行调用。
```
import cgitb
cgitb.enable()
```
### 构建CGI应用程序
+ 方法一
python2.x中直接执行
```
python -m CGIHTTPServer [port]
```
python3中用的http.server模块，包含一个基础服务器和三个请求处理程序类 BaseHTTPRequestHandler,SimpleHTTPRequestHandler,CGIHTTPRequestHandler;若没有提供端口，则默认使用8000
+ 方法二
从命令行中执行脚本，这种方法必须知道CGIHTTPServer的实际存储位置，POSIX系统中查找该包位置
```
In [28]: import sys,CGIHTTPServer

In [29]: sys.modules['CGIHTTPServer']
Out[29]: <module 'CGIHTTPServer' from '/usr/lib64/python2.7/CGIHTTPServer.pyc'>
In [31]: run /usr/lib64/python2.7/CGIHTTPServer.py  #由于是Ipython，用的run，正常用python来执行
Serving HTTP on 0.0.0.0 port 8000 ...
```
+ 方法三
使用-c选项运行由python语句组成的字符串。因此可以使用下面的方式导入CGIHTTPServer并执行其中的test()函数。
```
In [3]: !python -c "import CGIHTTPServer;CGIHTTPServer.test()"
Serving HTTP on 0.0.0.0 port 8000 ...
```
python3中
```
python3 -c 'from http.server import CGIHTTPRequestHandler,test;test(CGIHTTPRequestHandler)'
```
+ 创建快速脚本
前面通过-c选项导入并调用test()语句，将这些语句插入任意文件中，命名为cgihttpd.py文件（python2或3）
```
python cgihttpd.py 8080
```

这4种方式都会从当前目录下启动一个端口号为8000（或自行指定）的web服务器。将一些HTML文件放到启动服务器的目录中，在启动服务器目录下创建一个cgi-bin目录，放入python CGI 脚本。需要确保启动服务器的目录中有cgi-bin目录，同时确保其中有相应的.py文件，否则，服务器会将python文件作为静态文本返回，而不是执行这些文件。

### WSGI
CGI进程针对每个请求进行创建，用完就抛弃。发果应用程序接收千个请求，创建大量的语言解释器进和很快就会导致服务器停机。有两种方法可以解决这个问题，一是服务器集成，二是外部进程。
#### 服务器集成
服务器集成，也称为服务器API，这其中包括如Netscape服务器应用编程接口NSAPI和微软的因特网服务器ISAPI这样的商业解决方案。当前从20世纪90年代开始应用最广泛的是Apache HTTP Web服务器，这是一个开源的解决方案，通常称为Apache，拥有一个服务器API。所有这三个针对CGI性能的解决方案都将网关集成进服务器。换句话说，不是将服务器切分成多个语言解释器来分别处理请求，而是生成函数调用，运行应用程序代码，在运行过程中进行响应。服务器根据对应的API通过一组预先创建的进程或线程来处理工作。大部份可以根据需求进行调整，如提供压缩数据、安全、代理、虚拟主机等功能。
#### 外部进程
另一个解决方案就是外部进程。这是让CGI应用在服务器外部运行，当有请求进入时，服务器将这个请求传递到外部进程中，这种方式的可扩展性比纯CGI要好，因为外部进程存在时间很长，而不是处理完单个请求后就终止。使用最广泛的就是FastCGI，有了外部进程，就可以利用服务器API的好处，同时避免其缺点。比如在服务器外部运行就可以自由使用语言，应用程序的缺陷不会影响web服务器。FastCGI有Python实现，除此之外不有Apache的其他Python模块,如PyApache/mod_snkae/mod_python等。其中有些不再使用，这些模块加上纯CGI解决方案，组成了各种Web服务器API网关解决方案，以调用Python Web应用程序。
#### WSGI简介
WSGI不是服务器，也不是用于程序交互的API，更不是真实的代码，而只是定义的一个接口。用于处理日益增多的不同web框架，web服务器，及其他调用方式，如纯CGI、服务器API、外部进程。其目标是在web服务器和web框架之间提供一个通用的API标准，减少之间的互操作性并形成统一的调用方式。基本上所以基于python的web服务器都兼容WSGI。
