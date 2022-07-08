---
layout: post
published: true
title:  RabbitMQ官网阅读第三部份之Publish/Subscribe发布订阅
categories: [document]
tags: [rabbitmq,消息中间件]
---
* content
{:toc}

### 前言

前一章，我们创建了一个work queue，前面的假定条件是每个任务被投递到一个worker上面，这章我们要做的完全不同，我们将消息投递到多个消费者，这个就是著名的发布订阅；  
为了形象化的解释这种模式，我们将建一个简单的日志系统。它由两部份组成，一个是产生日志消息，另一个将从中接收并打印它们。  
在我们的日志系统中，每个接收系统都将拿到这个消息，这样，我们将运行一个receiver并直接将日志写到磁盘，另一个将接收并打印出来。   
本质上，published log 消息将会被广播到所有的接收端。

### Exchanges
在前面部份我们从发送和接收消息到queue和从queue中接收。  
回顾下前面章节中的内容：  
+ 一个producer就是一个应用，发送消息
+ 一个queue就是一个buffer，用来存储消息
+ 一个consumer就是一个应用，用来接收消息

在RabbitMQ消息模型中核心思想是生产者从不直接将消息发送到queue中，实际上，生产者并不知道消息是否被发送到queue上。  
相反，生产者只是将消息发送到exchange，一个exchange是一个简单的事情，一方面，它从生产者接收消息，另一方面，它将消息推到队列上。exchange必须知道如何处理接收到的消息，它是应该附加到一个特定的queue上面，还是附加到多个queue上面？或者还是应该被丢弃？ 这个规则被exchage的type所定义。

![exchage](/styles/images/exchanges.png)

exchage的type有： direct/topic/headers/fanout  
我们用最后一个fanout  
```python
channel.exchange_declare(exchange='logs',
                         exchange_type='fanout')
```
>列出exchanges：
```bash
sudo rabbitmqctl list_exchanges
```
在列出的结果里面，有需多amq.* exchanges和默认的未命名的exchange。它们是默认创建的，但目前你并不需要使用它们。  
**默认的exchange**  
前面几个章节中我们很少涉及exchage，但我们仍然可以给queue发送消息。这些能做到的原因是我们使用了默认的exchange，在定义时，我们用的空字符串。回想一下，我们之前如何定义消息的：
```python
channel.basic_publish(exchange='',
                      routing_key='hello',
                      body=message)
```
上面的exchage是空字符串是它的名字，这表明是默认或是无名的exchange：消息将会被路由到同名的routing_key的queue上。  

现在我们可以发布我们的exchage:
```python
channel.basic_publish(exchange='logs',
                      routing_key='',
                      body=message)
```

### Temporary queue临时队列
之前的章节你可能还记得我们给queue命名了，如hello,task_queue；能够给queue命名对我们很重要，我们需要将worker指定到相同的queue。尤其当我们需要在生产者和消费者之前共享队列时是至关重要的。  

但这个不是我们的场景，我们的场景是收取所有的日志消息，而不仅仅只是其中部份。我们只对当前传输的消息感兴趣，对以前老的消息不关心。为了解决这个问题，我们需要做两件事情：

+ 首先，无论何时连接到RabbitMQ,我们要一个新的空的queue。我们可以通过创建一个名字随机的queue来解决，甚至更好的，让server自动为我们选择一个随机的名字。我们可以通过不指定queue参数到queue_declare来实现。
```python
result = channel.queue_declare()
```
这样的话，result.method.queue包含了一个随机的队列名字。

+ 其次，一旦消费者连接关闭了，队列就会被删除，用exclusive 参数来实现它
```python
result = channel.queue_declare(exclusive=True)
```
关于exclusive标识和其他更多queue属性，你可能参考[更多](https://www.rabbitmq.com/queues.html)


### Bindings

![bindings](/styles/images/bindings.png)

我们已经创建了一个fanout exchange和队列，现在我们要告诉exchage发送一个消息到队列。exchange与queue之间的对应关系，我们称之为bindings  
```python
channel.queue_bind(exchange='logs',
                   queue=result.method.queue)
```
现在名为logs的exchage将会把消息放到我们的对列上。
> rabbitmqctl list_bindings可以列出存在的bindings

### CODE
将它们放到一起，如图：
![python-three-overall](/styles/images/python-three-overall.png)

+ emit_log.py
```python
#!/usr/bin/env python
import pika
import sys

connection = pika.BlockingConnection(pika.ConnectionParameters(host='localhost'))
channel = connection.channel()

channel.exchange_declare(exchange='logs',
                         exchange_type='fanout')

message = ' '.join(sys.argv[1:]) or "info: Hello World!"
channel.basic_publish(exchange='logs',
                      routing_key='',
                      body=message)
print(" [x] Sent %r" % message)
connection.close()
```

+ receive_logs.py:
```python
#!/usr/bin/env python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters(host='localhost'))
channel = connection.channel()

channel.exchange_declare(exchange='logs',
                         exchange_type='fanout')

result = channel.queue_declare(exclusive=True)
queue_name = result.method.queue

channel.queue_bind(exchange='logs',
                   queue=queue_name)

print(' [*] Waiting for logs. To exit press CTRL+C')

def callback(ch, method, properties, body):
    print(" [x] %r" % body)

channel.basic_consume(callback,
                      queue=queue_name,
                      no_ack=True)

channel.start_consuming()
```

运行

一个终端运行：
```bash
python receive_logs.py > logs_from_rabbit.log
```
另一个终端运行：
```bash
python receive_logs.py
```
运行产生日志程序：
```bash
python emit_log.py
```
查看绑定关系
```bash
sudo rabbitmqctl list_bindings
# => Listing bindings ...
# => logs    exchange        amq.gen-JzTY20BRgKO-HjmUJj0wLg  queue           []
# => logs    exchange        amq.gen-vso0PVvyiRIL2WoV3i48Yg  queue           []
# => ...done.
```
