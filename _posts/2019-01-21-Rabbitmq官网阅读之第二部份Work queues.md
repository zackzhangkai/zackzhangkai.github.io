---
layout: post
published: true
title:  RabbitMQ官网阅读之第二部份Work queues
categories: [document]
tags: [rabbitmq,消息中间件]
---
* content
{:toc}

### 前言

已经拖了很久了，放假之前一定要结束RabbitMQ，总不可能拖到明年吧。

### 内容

第一部份我们完成了从一个有名字的queue里面发送和接收消息，这一节我们要创建一个Work Queue，处理分布式的耗时的消费任务  
这样做的目的主要是为了避免集中式的任务，而且还需要等待它完成；相反，我们可以稍后来调度这个任务，我们把封装一个任务到一个队列中，在后台运行的一个工作进程，将会弹出这个任务，并最终执行它。当你运行多个works时，这个任务将会在它们中间进行共享。  
这个概念主要在web应用中用一个Http短请求处理复杂的任务时会很有。  
在前面的章节中，我们发送了一个"Hello,World"的消息。现在我们要创建一个字符串代表复杂的任务。我们没有真实的任务，如：图像识别或是pdf渲染，因此假装我们很忙，通过time.sleep()这个函数来模拟。我们用"."的个数来代替任务的复杂度，每个"."代表1s钟的任务，如：一个模拟的任务"Hello..."表示三秒钟。  
我们将前面的我们的send.py程序稍微改动下，下面这个程序将会把我们的任务封装到我们的queue中，名为new_task.py

```python
import sys

message = ' '.join(sys.argv[1:]) or "Hello World!"
channel.basic_publish(exchange='',
                      routing_key='hello',
                      body=message)
print(" [x] Sent %r" % message)
```

同时，我们之前老的receive.py同样需要做下修改，它需要模拟出每个"."代表一秒钟。它将会从queue中取出消息，并且执行这个任务。我们名为"task.py"  

```python
import time

def callback(ch, method, properties, body):
    print(" [x] Received %r" % body)
    time.sleep(body.count(b'.'))
    print(" [x] Done")
```

### Round-robin 分发  

用Task Queue的好处的优点是可以并行处理任务，如果我们可以让任务积压，我们可以同时启动更多的消费者，因此我们可以同时从队列里拿消息，那么具体如何做呢？  
我们启动三个终端，两个运行work.py脚本，这两个就是我们的两个消费者。

```bash
# shell 1/2
python worker.py
# => [*] Waiting for messages. To exit press CTRL+C
```

另外一个终端，我们运行我们的生产者

```bash
# shell 3
python new_task.py First message.
python new_task.py Second message..
python new_task.py Third message...
python new_task.py Fourth message....
python new_task.py Fifth message.....
```

我们看下，我们的消费者终端

```bash
# shell 1
python worker.py
# => [*] Waiting for messages. To exit press CTRL+C
# => [x] Received 'First message.'
# => [x] Received 'Third message...'
# => [x] Received 'Fifth message.....'
```

另开一个终端

```bash
# shell 2
python worker.py
# => [*] Waiting for messages. To exit press CTRL+C
# => [x] Received 'Second message..'
# => [x] Received 'Fourth message....
```

默认的，RabbitMQ将会平均的分配这些消息，这个方式就称为Round Robin.

###  Message acknowledgment

处理任务时耗时的，如果其中一个消费者处理了一个很长的任务，然后只处理了一半就宕掉了，那么会发生什么呢？  
按照我们当前的代码逻辑，一旦RabbitMQ将消息发送给了消费者，它马上会将这个消息标记为删除(deletion)。那么在这种场景下，如果worker宕掉了，我们的消息将会丢失；同时，我们将会丢掉所有已经发送给这个worker，但是未被这个worker处理的消息。

但是我们不想丢失任何一个任务，我们要求如果一个消费者worker宕掉了，我们要求将这个消息发送给另外一个worker来处理。

为了保证我们所有的消息不丢失，RabbitMQ支持消息应答（Message acknowledgments）机制。消费者将会告诉RabbitMQ，哪个消息已经接收到了，已经处理，这时候，RabbitMQ才会来删除这个消息。
如果一个消费者宕掉了，即它的channel关闭了、连接关闭了或者是TCP连接丢失了，它将不会给RabbitMQ发送应答ack,此时，RabbitMQ知道这个消息未被完全处理，并且将会重新将这个消息放进queue中。如果同时有其他的消费者在线，它将会将这个消息发送到另一个消费者里进行处理。通过这种方式，能保证消息不被丢失，即使是有的worker突然宕掉。
Message acknowledgement默认是开启的，在前面例子里面我们手动将它关了 no_ack=True ,现在我们开启应答

```python
def callback(ch, method, properties, body):
    print " [x] Received %r" % (body,)
    time.sleep( body.count('.') )
    print " [x] Done"
    ch.basic_ack(delivery_tag = method.delivery_tag)

channel.basic_consume(callback,
                      queue='hello')
```

通过这个代码，我们能保证我们的消息不丢失，即失在运行work.py时，我们按了ctrl+c终断了我们的程序。  
acknowledgement必须发送到相同的channel，如果用不同的channel来应答，那么将会产生channel-level协议的exception，具体的请参考[更多](doc guide on confirmations)

*最常见的错误就是忘记了应答，即忘记了写basic_ack，虽然这个错误很容易犯，但后果是很严重的。如果worker宕掉了，消息同样会被重新投递，但是随机重新投递，RabbitMQ将会吃掉更多更多的内存，因为它不会释放任何没有应答的消息。为了能够debug这个错误，你可以通过rabbitmqctl来查询字段为messages_unacknowledged的消息。*

```bash
sudo rabbitmqctl list_queues name messages_ready messages_unacknowledged
```

### Message durability消息持久化

我们已经保证了如何不丢消息，即使worker宕掉，但是我们的消息仍有可能丢失，如果是RabbitMQ服务宕掉。  
当RabbitMQ退出或是崩溃时，它将不记得队列和消息，除非你告诉它。因此我们要做两件事情来避免这种意外：一是要同时将消息message和queue标记为持久性durable

```python
channel.queue_declare(queue='hello', durable=True)
```

虽然这样设置是正确的，但最终不会成功，我们之前已经定义了一个名为hello的非持久化的queue,RabbitMQ不允许你用不同的参数重新定义一个已经存在的queue，如果有程序尝试这样做，它会给你返回一个错误告诉你这个问题；因此我们可以定义一个不同名字的queue

```python
channel.queue_declare(queue='task_queue', durable=True)
```

queue_declare需要同时在生产者和消费者定义。

只有这样，我们才能保证，task_queue不会丢失，即使我们的RabbitMQ服务重启。现在我们要将我们的消息标记为持久性，通过设置delivery_mode的值。

```python
channel.basic_publish(exchange='',
                      routing_key="task_queue",
                      body=message,
                      properties=pika.BasicProperties(
                         delivery_mode = 2, # make message persistent
                      ))
```

*消息持久化并不保证消息不会丢失，虽然消息持久化是告诉RabbitMQ将消息持久化到磁盘，它同样有一个消息从接收到保存的时间，而且它也不是把每个消息持久化到磁盘中-它也许只是将它保存到缓存并没有真正写到磁盘。它虽然并不是绝对的强壮，但对于我们的简单的任务队列，它绰绰有余。如果你要求一个更强壮的机制，你可以使用[生产者应答](https://www.rabbitmq.com/confirms.html) (publishe confirms)*

### Fair dispatch公平的分发

你也许注意到了，这种分发机制并不是我们想要的，比如有这种场景：有两个worker，有些一消息很得、有些消息很轻，然后有些worker一直持续的忙碌，而另外一个总是很闲；然而RabbitMQ并不知道这种情况，它仍旧像以往那样分发消息。

造成这种问题的原因是RabbitMQ当有消息进入时，它只是分发而已，它并不关心消费者消息未应答的数量。

为了避免这种情况：我们可以使用basic.qos方法设置prefetch_count=1。它的意思是不要每次给一个worker多个消息。换句话说，它的意思是，在新的消息处理并应答之前不要给它分配消息。消息将会给到空闲的worker

```python
channel.basic_qos(prefetch_count=1)
```

*如果你的worker都很忙，那么你的queue将很会填满，你需要时刻观察这个情况，在有必要的情况下增加worker*

### 代码

new_task.py

```python
#!/usr/bin/env python
import pika
import sys

connection = pika.BlockingConnection(pika.ConnectionParameters(host='localhost'))
channel = connection.channel()

channel.queue_declare(queue='task_queue', durable=True)

message = ' '.join(sys.argv[1:]) or "Hello World!"
channel.basic_publish(exchange='',
                      routing_key='task_queue',
                      body=message,
                      properties=pika.BasicProperties(
                         delivery_mode = 2, # make message persistent
                      ))
print(" [x] Sent %r" % message)
connection.close()
```

work.py

```python
#!/usr/bin/env python
import pika
import time

connection = pika.BlockingConnection(pika.ConnectionParameters(host='localhost'))
channel = connection.channel()

channel.queue_declare(queue='task_queue', durable=True)
print(' [*] Waiting for messages. To exit press CTRL+C')

def callback(ch, method, properties, body):
    print(" [x] Received %r" % body)
    time.sleep(body.count(b'.'))
    print(" [x] Done")
    ch.basic_ack(delivery_tag = method.delivery_tag)

channel.basic_qos(prefetch_count=1)
channel.basic_consume(callback,
                      queue='task_queue')

channel.start_consuming()
```

至此完成Rabbitmq官网的workqueue工作模式的阅读。
