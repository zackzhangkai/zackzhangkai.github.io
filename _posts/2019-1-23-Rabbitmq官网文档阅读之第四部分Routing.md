---
layout: post
published: true
title:  Rabbitmq官网文档阅读之第四部分Routing
categories: [document]
tags: [rabbitmq,消息中间件]
---
* content
{:toc}

### 前言
接触消息队列很长时间，说起中间件，曾经面试时也在这个地方踩过坑；消息队列的作用主要是解决软件分布式异步通信，解耦系统依赖的作用；Rocketmq是阿里基于Kafa开源出来，只能用java语言；Rocketmq则可用多个语言，包括python；openstack中的rpc的通信则是基于rabbitmq；最近花了一段时间研究rabbitmq，主要是通过官方文档及github官方仓库源码，结合本地实验验证理解；

### Routing
本地rabbitmq安装并启动；<br>
端口：5672 15672 25672<br>
用pika python client<br>
实验目的：只订阅部份消息；如只将critical error日志写进文件，同时将所有的日志输出到控制台

### Bindings
```python
channel.queue_bind(exchange=exchange_name,queue=queue_name)
```
binding将exchange与queue的连接关系；即queue可以选择订阅exchange的哪些消息;<br>
可以跟该binding加一个额外的参数routing_key；记得这与basic_publish里面的binding key不一样，binding_key是生产者端，routing_key是消费者端；
```python
channel.queue_bind(exchange=exchange_name,queue=queue_name,routing_key='black')
```

### Direct exchange
exchange的几种类型：fanout,topic,headers,direct<br>
fanout为广播，向所有的queue发送消息；<br>
我们将使用direct,它的特点是消息进到queue时会让binding key与routing key进行匹配；<br>
在消费者端创建bing时，可以创建多个相同的routing key，那这个对应的消息就可以发送到多个queue里面；这就有点类似fanout<br>

生产者：<br>
首先创建一个exchange
```python
channel.exchange_declare(exchange='direct_logs',
                         exchange_type='direct')
```
准备发送
```python
channel.basic_publish(exchange='direct_logs',
                      routing_key=severity,
                      body=message)
```
severity：'info'/'warning'/'error'  

消费者：<br>

```python
result = channel.queue_declare(exclusive=True)
queue_name = result.method.queue

for severity in severities:
    channel.queue_bind(exchange='direct_logs',
                       queue=queue_name,
                       routing_key=severity)
```               

### CODE

1. emit_log_direct.py（生产者）

```python
#!/usr/bin/env python
import pika
import sys

connection = pika.BlockingConnection(pika.ConnectionParameters(host='localhost'))
channel = connection.channel()

channel.exchange_declare(exchange='direct_logs',
                         exchange_type='direct')

severity = sys.argv[1] if len(sys.argv) > 1 else 'info'
message = ' '.join(sys.argv[2:]) or 'Hello World!'
channel.basic_publish(exchange='direct_logs',
                      routing_key=severity,
                      body=message)
print(" [x] Sent %r:%r" % (severity, message))
connection.close()
```

2. receive_logs_direct.py（消费者）

```python
#!/usr/bin/env python
import pika
import sys

connection = pika.BlockingConnection(pika.ConnectionParameters(host='localhost'))
channel = connection.channel()

channel.exchange_declare(exchange='direct_logs',
                         exchange_type='direct')

result = channel.queue_declare(exclusive=True)
queue_name = result.method.queue

severities = sys.argv[1:]
if not severities:
    sys.stderr.write("Usage: %s [info] [warning] [error]\n" % sys.argv[0])
    sys.exit(1)

for severity in severities:
    channel.queue_bind(exchange='direct_logs',
                       queue=queue_name,
                       routing_key=severity)

print(' [*] Waiting for logs. To exit press CTRL+C')

def callback(ch, method, properties, body):
    print(" [x] %r:%r" % (method.routing_key, body))

channel.basic_consume(callback,
                      queue=queue_name,
                      no_ack=True)

channel.start_consuming()
```

生产者端运行：

```bash
python emit_log_direct.py error "Run. Run. Or it will explode."
python emit_log_direct.py info
python emit_log_direct.py warning
```

消费者端运行：

只保存'warning'和'error'（不包含'info'）日志到文件
```bash
python receive_logs_direct.py warning error > logs_from_rabbit.log
```

将所有的日志全部打印到console
```bash
python receive_logs_direct.py info warning error
```

download [emit_log_direct.py](/styles/emit_log_direct.py)
download [receive_logs_direct.py](/styles/receive_logs_direct.py)
