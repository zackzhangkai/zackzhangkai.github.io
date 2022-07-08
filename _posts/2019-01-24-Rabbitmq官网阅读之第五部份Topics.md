---
layout: post
published: true
title:  Rabbitmq官网阅读之第五部份Topics
categories: [document]
tags: [rabbitmq,消息中间件]
---
* content
{:toc}

### 前言

上一章验证了不用fanout而是通过direct exchange，实现了通过日志级别来接收消息<br>
这一章我们要实现不仅仅是根据日志级别，同时也要根据日志来源来接收消息，这个有点于类似unix的根据日志级别（info/warning/critical）和场所（auth/cron/kern）来记录的syslog<br>
通过这个组合，我们就能实现记录所有的cron的critical日志，同时记录kern的所有日志，这就极大的增强了系统的多样性<br>
这个就是我们本章要实现的，可以用topic exchange来实现

### Topic exchange
我们回忆下exchange有几种，mq是怎样工作的？<br>
mq的工作流程：<br>
首先分为生产者和消费者，生产者：调用pika库，建立一个channel--创建exchange,指定exchange name exchange_type--定义消息body--发送消息publish,指定exchange,binding key,body,每publish一次相当于给发送了一次消息，而binding key相当于给这个消息打了一个标签，在消费者那端接收时，可以跟routing key对应来过滤消息;<br>
消费者：同理调用pika库后，建立一个channel--创建一个exchange,指定exchange type,name--创建queue,指定queue name，queue的属性，如是否持久化，是否消息确认，还是完成消息后删除(exclusive)--创建binding，让queue选择从哪个exchange来获取消息，指定exchange name跟queue name的对应关系，并明确收哪些消息(routing_key)--至此一切准备好接收消息了，但是还要想想，消息接收后要怎么处理这个消息，主要是就是从消费者那端拿到的消息body,这个是生产者要传给我们的，那我们就需对这个body进行操作，如何操作，此时就需要定义一个函数，我们叫它回调函数--最后，就只剩创建消费者了，channel.basic_consume(callback,queue),这里要注意，只是传入函数名，函数的调用在这个消费者里自动调用，指定queue name，另外还要定义是否要消息确认，默认是要消息确认的，如果不用消息确认就加上no_ack参数；-- 最后一步，创建完的消费者，让他执行消费channel.start_consuming()就行了。<br>

那如果exchange是用topic的话，则不能简单的使用上述的routing key，毕竟它只是一个string字符串，太单一；现在你应该猜到了，它应该是多维的，它们定义了消息的属性，相当于给这些消息加上一些标识，如 “stock.usd.nyse” , "nyse.vmw","quick.orange.rabbit",只需要注意不要超过255个字节就行。同理，因binding key 与 routing key是成对出现的，因此，routing key要是相同的形式；如果单单从这看的话，topic跟direct并没有太大区别；但是既然不一样，肯定有不一样的地方，区别主要在如下两个地方：<br>
**\***:星号可以代表一个词<br>
**\#**：井号代表零个或多个词<br>
如图：<br>
![](/styles/images/rabbitmq-topics1.png)
说白了，就是topics跟direct不一样的地方在于它的binding_key加入了正则匹配，这样消费者在接收它时，可以采用正则匹配的方式来攻取消息，增加了消息的组合形式；<br>

### CODE
1. emit_log_topic.py

```python
#!/usr/bin/env python
import pika
import sys

connection = pika.BlockingConnection(pika.ConnectionParameters(host='localhost'))
channel = connection.channel()

channel.exchange_declare(exchange='topic_logs',
                         exchange_type='topic')

routing_key = sys.argv[1] if len(sys.argv) > 2 else 'anonymous.info'
message = ' '.join(sys.argv[2:]) or 'Hello World!'
channel.basic_publish(exchange='topic_logs',
                      routing_key=routing_key,
                      body=message)
print(" [x] Sent %r:%r" % (routing_key, message))
connection.close()
```

2. receive_logs_topic.py

```python
#!/usr/bin/env python
import pika
import sys

connection = pika.BlockingConnection(pika.ConnectionParameters(host='localhost'))
channel = connection.channel()

channel.exchange_declare(exchange='topic_logs',
                         exchange_type='topic')

result = channel.queue_declare(exclusive=True)
queue_name = result.method.queue

binding_keys = sys.argv[1:]
if not binding_keys:
    sys.stderr.write("Usage: %s [binding_key]...\n" % sys.argv[0])
    sys.exit(1)

for binding_key in binding_keys:
    channel.queue_bind(exchange='topic_logs',
                       queue=queue_name,
                       routing_key=binding_key)

print(' [*] Waiting for logs. To exit press CTRL+C')

def callback(ch, method, properties, body):
    print(" [x] %r:%r" % (method.routing_key, body))

channel.basic_consume(callback,
                      queue=queue_name,
                      no_ack=True)

channel.start_consuming()
```

运行：<br>
消费者端:<br>
接收所有logs
```bash
python receive_logs_topic.py "#"
```
从kern来的地方接收日志
```bash
python receive_logs_topic.py "kern.*"
```
接收所有的critical日志
```bash
python receive_logs_topic.py "*.critical"
```
接收来自kern 和 critical日志组合
```bash
python receive_logs_topic.py "kern.*" "*.critical"
```

生产者端：<br>
创建一个包含 ‘kern.critical’的routing key
```bash
python emit_log_topic.py "kern.critical" "A critical kernel error"
```

### 代码下载
you can download [receive_logs_topic.py](/styles/receive_logs_topic.py)
you can download [emit_log_topic.py](/styles/emit_log_topic.py)
