---
layout:       post
title:        "chapter4-python-多线程编程"
date:         2017-11-30 12:00:00
categories: document
tag: python
---

* content
{:toc}

### threading模块
thrading模块支持守护线程：守护线程一般是一个等待客户端请求的服务器。如果没有客户端的请求，守护线程就是空闲的。如果把一个线程设置成一个守护线程，就表示这个线程是不重要的，进程退出时不需要等待这个线程执行完成。要将一个线程设置为守护线程，需要在启动之前执行如下赋值语句 thread.daemon=True整个Python程序(可以解读为主线程)将在所有的非守护线程退出后才会退出，换句话说，就是没有剩下存活的非守护线程时。
#### Thread类对象
threading的Thread类是主要执行对象

属性|描述
-|-
thread对象数据属性|
name|线程名
ident|线程的描述符
daemon|布尔标志，表示这个线程是否是守护线程
Thread对象方法|
_init_(group=None,target=None,name=None,args=(),kwargs={},verbose=None,daemon=None)|实例化一个线程对象，需要有一个可调用 的target，以及其参数args或kwargs。daemon的值将会设定thread.daemon属性/标志。

使用Thread类，可以有很多方法来创建线程。常规三种：
+ 创建Thread的实例，传给他一个函数
+ 创建Thread的实例，传给它一个可调用的类实例
+ 派生Thread的子类，并创建子类的实例

>倾向使用最后一种

```
#!/usr/bin/env python
import threading
from time import sleep,ctime
loops = [4,2]

def loop(nloop,nsec):
  print "start loop",nloop,"at:",ctime()
  sleep(nsec)
  print "loop",nloop,"done at:",ctime()

def main():
  print "starting at:",ctime()
  threads=[]
  nloops=range(len(loops))

  for i in nloops:
    t = threading.Thread(target=loop,args=(i,loops[i]))
    threads.append(t)

  for i in nloops:
    threads[i].start()  #调用start()函数时，线程开始执行

  for i in nloops:
    threads[i].join()#调用join()等待线程执行完成后退出

  print "all DONE at:",ctime()

if __name__ == "__main__":
  main()
```
执行结果
```
[root@i612-devopsyw-1 tmp]# python mtsleepC.py
starting at: Thu Nov 30 15:43:36 2017
start loop 0 at: Thu Nov 30 15:43:36 2017
start loop 1 at: Thu Nov 30 15:43:36 2017
loop 1 done at: Thu Nov 30 15:43:38 2017
loop 0 done at: Thu Nov 30 15:43:40 2017
all DONE at: Thu Nov 30 15:43:40 2017
```
