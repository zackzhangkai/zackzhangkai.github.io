---
layout:       post
title:        "chapter9-web客户端和服务端"
date:         2017-08-11 12:00:00
categories: document
tag: python
---

* content
{:toc}

### 简介
web应用遵循客户端/服务器架构，web客户端是浏览器，即允许用户在万维网上查询文档的应用程序。另一边是web服务器端，指的是运行在信息提供商的主机上的进程。这些服务器等待客户端及其文档的请求，进行相应的处理，并返回相关的数据。服务器持续运行，客户端的活动以单个事件划分，一旦完成一个客户请求，这个服务事件就停止了。web客户端可以向服务器发出各种不同的请求。这些请求可能包括获得一个用于查看的网页视图，或者提交一个包含待处理数据的表单。web服务器端首先处理请求，然后会以特定的格式HTML等返回给客户端浏览。web客户端和服务器端端交互需要用特定的语言，即web交互需要用到的标准协议，称为HTTP(超文本传输)。HTTP是TCP/IP的上层协议，这意味着HTTP协议依靠TCP/IP来进行低层的交流工作。它的职责不是发送或者传递消息(TCP/IP协议来处理这些)，而是通过发送、接受HTTP消息来处理客户端的请求。客户端可以随时发送新的请求，但是新的请求会处理成独立的服务请求。由于每个请求缺乏上下文，因此你可能注意到有些URL中含有很长的变量和值，这些将作为请求的一部分，以提供一些状态信息。另一种方式是使用“cookie”，即保存在客户端的客户状态信息。值得一提的是，在因特网上传输的数据当中，其中一部份会比较敏感。而在传输过程中，默认没有加密服务，所以标准协议直接将应用程序发送过来的数据传输出去。为了对传输数据进行加密，需要在普通的套接字上添加一个额外的安全层，称为安全套接字层secure socketlayer，简称SSL，用来创建一个套接字，加密通过该套接字传输的数据。开发者可以决定是否使用这个额外的安全层。
### Python web客户端工具
有一点需要记清楚，浏览器只是web客户端的一种。 任何一个web服务端发送请求来获得数据的应用程序都是“客户端”。使用urllib模块下载或者访问web上信息的应用程序(urllib.urlopen()或者urllib.urlretrieve()就是简单的web客户端
#### URL统一资源定位符
简单的网页浏览需要用到URL(统一资源定位符)的web地址。这个地址用来在web上定位一个文档，或者调用一个CGI程序来为客户端生成一个文档。URL是URI(统一资源标识符)的一部分。一个URL是一个简单的URI,它使用已有的协议方案，如HTTP、ftp等作为地址的一部分。
```
#URL的格式：
prot_sch://net_loc/path;params?query#frag
```
+ prot_sch 网络协议或下载方案
+ net_loc 服务器所在地(也许含有用户信息)，可进一步拆分成 user:passwd@host:port,用户名和密码只有在使用ftp连接时才有可能用到，而即使是使用FTP，大多数的连接都是匿名的，这时不需要用户名和密码
+ path 使用斜杠/分割的文件或CGI应用的路径
+ params 可选参数
+ query 连接符&分割的一系列键值对
+ frag 指定文档内特定锚的部分

python支持两种不同的模块，两者分别以不同的功能和兼容性来处理URL。
#### urlparse模块
+ urlparse.urlparse()
```
#语法
(urlstr,defProtSch=None,allowFrag=None)
```
urlparse()将urlstr解析成一个6元组(pro_sch,net_loc,path,params,query,frag),前面已经描述了这每个组件。如果urlstr中没有提供默认的网络协议或下载方案，defProtSch会指定一个默认的网络协议。allowFrag标识一个URL是否允许使用片段。下面是一个给定URL经urlparse()后的输出。
```
In [25]: urlparse.urlparse('https://www.python.org/doc/FAQ.html')
Out[25]: ParseResult(scheme='https', netloc='www.python.org', path='/doc/FAQ.html', params='', query='', fragment='')
```
+ urlparse.urlunparse()
与urlparse()恰好相反，其将经urlparse()处理的URL生成urltup这个6元组拼接成URL并返回。因此
```
urlunparse(urlparse(urlstr)) == urlstr
```
+ urlparse.urljoin()
在需要处理多个相关的URL时我们就需要使用urljoin()的功能了，例如，一个Web页中可能会产生一系列页面URL。

#### urllib模块/包
python2中有urllib、urlparse、urllib2。python3中全部整合进urllib单一包中。urllib和urlib2中的内容整合进了urllib.request模块中，urlparse整合进了urllib.parse中。python3中的urllib包还包括response、error和robotparse这些子模块。
+ urllib.urlopen()，打开一个给定的URL字符串表示的Web连接，并返回文件类型的对象。
```
urlopen(urlstr,postQueryData=None)
```
urlopen()打开urlstr所指向的url，如果没有给定协议或者下载方案scheme，或者传入了一个file方案，urlopen()会打开一个本地文件。一旦连接成功，urlopen()将会返回一个文件类型对象，就像在目标路径下打开一个可读文件。如果文件对象是f，那么句柄会支持一些读取内容的方法，如f.read()，f.readline(),f.readlines(),f.close()，f.fileno()
