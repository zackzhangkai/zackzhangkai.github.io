---
layout:       post
title:        "chapter2-socket网络编程"
date:         2017-07-07 12:00:00
categories:    document
tag: python
---

* content
{:toc}

### 套接字
套接字：通讯端点，网络化的应用在开始任何通讯之前都必需要创建套接字。就像电话的插口一样，没有它就没法通讯。一开始套接字被设计用在同一台主机上多个应用程序之间的通讯。这也被叫做进程间的通讯，或叫IPC。套按字有两种，分别是基于文件型和基于网络型的。Unix套接字是第一个套接字家族，家族名“AF_UNIX”也叫AF_LOCAL，表示地址家族：UNIX；python使用AF_UNIX。另一种套接字是基于网络的，它有自己的家族名：AF_INET，或叫地址家族：Internet。
### 套接字地址：主机与端口
若把套接字比如电话插口--通讯的最底层结构，那主机与端口就像区号与电话号码的一对组合。有了能打电话的硬件还不够，你得知道打给谁，往哪打。一个Internet地址就是一个主机与端口组成。而且对方要有人听，才行。
>合法的端口号范围为0到65535。其中小于1024的端口号为系统保留端口。Unix操作系统保留的端口号及其对应的服务/协议/套接字类型可以通过/etc/services文件获得

### 面向连接与无连接
+ 面向连接
无论你使用哪一种地址家庭，套接字的类型只有两种。一种是面向连接的套接字，即在通讯之前一定要建立一个连接，就像跟朋友打电话时那样。这种通讯方式也被称为“虚电路”或“流套接字”。面向连接的通讯方式提供了顺序的，可靠的，不会重复的数据传输，而且也不会被加上数据边界。这也意味着，每一个要发送的信息，可能会被拆分成多份，每一份都会不多光少地正确到达目的地。然后重新按顺序拼装起来，传给正在等待的应用程序。实现这种连接的主要协议就是传输控制协议TCP。要创建TCP套接字就得在创建的时候指定套接字类型为SOCK_STREAM。TCP套接字采用SOCK_STREAM这个名字，表达了它做为流套接字的特点。由于这些套接字使用Internet协议IP来查找网络中的主机，这样形成的整个系统，一般会由这两个协议来提及，即TCP/IP。
+ 无连接
与虚电路完全相反的是数据报型的无连接套接字。这意味着，无需建立建立连接就可以进行通讯。但这时，数据到达的顺序，可靠性及数据不重复就无法保证了。数据报会保留数据边界，这就表示，数据不会像面向连接的协议那样被拆分成小块。使用数据报来传输数据就像邮政服务一样。邮件和包裹不一定会按它们发送的顺序到达。事实上，它们还有可能根本到不了。而且由于网络的复杂性，数据还可能被重复传送。既然数据报有这么多缺点，为什么还要使用它呢，一定是有什么方面胜过流套接字。由于面向连接套接字要提供一些保证，以及要维持电路连接，这都是很重的额外负担。数据报没有这些负担，所以它更“便宜”。通常能提供更好的性能，更适合某些应用场合。实现这种连接的主要协议就是用户数据报协议UDP，要创建UDP套接字就得在创建的时候指定套接字类型为SOCK_DGRAM。SOCK_DGRAM来源于datagram(数据报)，同样在Internet中通过IP查找主机，即UDP/IP。

### python中的网络编程
主要利用socket()模块，模块中的socket()函数被用来创建套接字。套接字也有自己的一套函数来提供基于套接字的网络通信。
#### socket()模块
socket.socket()来创建套接字。语法
```
socket(socket_family,socket_type,protocol=0)
```
+ socket_family: AF_UNIX 或 AF_INET
+ socket_type: SOCK_STREAM 或 SOCK_DGRAM
+ protocol: 一般不填，默认值0
```
#创建一个TCP/IP的套接字，你要这样调用socket.socket():
tcpSock = socket.socket(socket.AF_INET,socket.SOCK_STREAM)
#同样地，创建一个UDP/IP的套接字
udpSock = socket.socket(socket.AF_INET,socket.SOCK_DGRAM)
```
当我们创建套接字对象后，所有的交互都将通过对该套接字对象的方法调用进行。
#### 套接字对象(内建)方法
+ s.blind() 绑定地址(主机，端口号对)到套接字
+ s.listen() 开始TCP监听
+ s.accept() 被动接受TCP客户的连接，(阻塞式)等待连接的到来客户端套接字函数
+ s.connect() 主动初始化TCP服务器连接
+ s.connetct_ex() connetct()函数的扩展版本，出错时返回出错码，而不是抛异常公共用途的套接字函数
+ s.recv() 接受TCP数据
+ s.send() 发送TCP数据
+ s.sendall() 完整发送TCP数据
+ s.recvfrom() 接收UDP数据
+ s.sendto() 发送UDP数据
+ s.getpeername() 连接到当前套接字的远端的地址
+ s.getsockname() 当前套接字地址
+ s.getsockopt() 返回指定套接字的参数
+ s.setsockopt() 设置指定套接字的参数
+ s.close() 关闭套接字
> 提示：在运行网络应用程序时，最好在不同的电脑上执行服务器和客户端的的程序
#### 创建一个TCP服务器
伪代码
```
import socket
ss = socket.socket()  #创建套接字
ss.bind() #将地址绑定到套接字上
ss.listen() #监听连接
inf_loop: #服务器无限循环
cs = ss.accept() #接受客户的连接，默认情况下，accept()函数是阻塞式的，即程序到来之前会处理挂起状态。
comm_loop: #通讯循环
cs.recv()/cs.send() #对话
cs.close() #关闭客户套接字,服务器会继续等待下一个客户的连接
ss.close() #关闭服务器套接字(可选)
```
代码如下

```python
#!/usr/bin/env python
#_*_coding:UTF8_*_
from socket import *
from time import ctime
#
#HOST为空表示bind()函数可以绑定在所有有效地址上。
HOST = ''
#随便一个未被占用的端口号
PORT = 21567
#缓冲大小设定为1k，根据实际网络情况和应用的需要来修改这个大小
BUFSIZ = 1024
ADDR = (HOST,PORT)
#网络、tcp
tcpSerSock = socket(AF_INET,SOCK_STREAM)
tcpSerSock.bind(ADDR)
#listen()函数的参数只是表示最多允许多少个连接同时连进来，后来的连接就会被拒绝掉
tcpSerSock.listen(5)
#
#无限循环
while True:
    print 'waiting for connection...'
    #被动等待连接的到来
    tcpCliSock,addr = tcpSerSock.accept()
    #开始会话
    print '...connected from:',addr
    while True:
        data = tcpCliSock.recv(BUFSIZ)
        #如果消息为空，表示客户端已经退出，那就再去等待下一个客户的连接
        if not data:
            break
        #得到消息后，加上时间戳后返回
        tcpCliSock.send('%s:%s\n' % (ctime(),data))
        tcpCliSock.close
    tcpSerSock.close()
```

通过 telnet就可以连接测试访问

#### 创建一个TCP客户端
伪代码
```
cs = socket() #创建客户套接字
cs.connect() #尝试连接服务器
comm_loop: #通讯循环
cs.send()/cs.recv() #对话(发送/接收)
cs.close() #关闭客户套接字
```
所有的套接字都可以由socket.socket()函数创建。在客户有了套接字后，马上就可以调用connect()函数去连接服务器。连接建立后，就可以与服务器开始对话了。对话结束后，客户就可以关闭套接字，结束连接。
```
#!/usr/bin/env python
#_*_coding:UTF8_*_
from socket import *
from time import ctime

#HOST为空表示bind()函数可以绑定在所有有效地址上。
HOST = '127.0.0.1'
#随便一个未被占用的端口号
PORT = 21567
#缓冲大小设定为1k，根据实际网络情况和应用的需要来修改这个大小
BUFSIZ = 1024
ADDR = (HOST,PORT)
#网络、tcp
tcpCliSock = socket(AF_INET,SOCK_STREAM)
tcpCliSock.connect(ADDR)

#无限循环
while True:
    data = raw_input('>:')
    if not data:
        break
    #得到消息后，加上时间戳后返回
    tcpCliSock.send(data)
    data = tcpCliSock.recv(BUFSIZ)
    if not data:
        break
    print data
    tcpCliSock.close
```

### 通信python提供两种方法:socket对象及文件类对象
#### socket对象
+ send()
+ sendto()
+ recv()
+ recvfrom()
#### 文件类对象
+ read()
+ write()
+ readline()

#### 异常
+ 与一般I/O和通信有关的: socket.error
+ 与查询地址信息有关的socket.gaierror
+ 与其他地址错误有关的socket.herror
+ 与在一个socket上调用settimeout()后，处理超时有关的socket.timeout
在connetct(0)的调用时，如果程序可以解决把主机名转换成IP地址的问题，可能产生两种错误，如果主机名不对则会产生socket.gainerror，如果连接远程主机有问题则会产生socket.error

### 必需的参数
+ port： 描述了端口号，或者定义在/etc/services的名字，这个端口号是服务器应该侦听的，例如 80,http
+ type: 若是tcp，则type是SOCK_STREAM;若是udp，则是dgram
+ protocol: tcp或者是udp
+ invocationtype: 对于tcp服务器，invocationtype应该是nowait;对于udp，如果服务器连接远程机器并为来自不同的机器的信息包请求一个新的进程来处理，那么使用nowait。如果UDP在它的端口上处理所有的信息包，直到它终止，那么应该使用wait。
+ username： username 指定了服务器应该在哪个用户下运行
+ path: path是指服务器的完整路径
+ programname: 表示程序的名字，就像在sys.argv[0]中传递的那样
+ arguments: 这个参数是可选的，如果有，则在服务器的脚本中以sys.argv[1:]显示

>  注意，因为socket在连接时，只能有一个进程来连接，若每每个socket开一个进程，势必则会将内存占满；故需要配置xinetd

### syslog模块
python提供了一个可以作为系统syslog程序接口的syslog模块。在开始记录信息之前，需调用openlog()函数来初始化syslog的接口，
```
openlog(ident[,logopt[,facility]])
```
