---
layout: post
published: true
title:  Rabbitmq官网阅读之第六部份RPC
categories: [document]
tags: [rabbitmq,消息中间件]
---
* content
{:toc}

### 前言

终于到RPC了，rabbitmq看了一周，今天要结束了，终于也迎来了重头戏rpc，说起通信协议有这么几种：Restful
、rpc、grpc（谷歌开源）；对外的API一般是用的Restful、对内一般是用的rpc；那么为什么呢？效率or安全？还是啥？

### RPC
前面第二部份我们讲了如何用多个worker来消费，但是如果我要在远端执行一个函数方法并要求返回执行结果，那该怎么办呢？  
这就是远程结果调用Remote Procedure Call即RPC  
本章我们将利用RabbitMQ来找建一个RPC系统，一个client和一个server;因为我们无需消费者来执行任务，因此我们只需要RPC服务返回我们斐波那契（Fibonacci）数字就行  
1. Client interface<客户端接口>  
    为了示例清楚RPC服务是怎样工作的，我们将编写一个call方法，来发送RPC请求，并且阻塞至到收到应答   
    ```python
    fibonacci_rpc = FibonacciRpcClient()
    result = fibonacci_rpc.call(4)
    print("fib(4) is %r" % result)
    ```
    >尽管RPC经常被使用，同时它也经常被人诟病，因为如果开发者不注意以下两点的话就会有问题：一个是该function是否是本地方法，另一个是如果它是一个慢RPC；如果是以上所述情况的话，就会导致一个不可预测的系统，并且会增加很多不必要的debugging;与一般的简单的软件不同，如果错用RPC，它会导致不可维护的冗余的代码；为了避免以上情况，有以下建议：
        - 搞清楚将要被调用的方法是本地方法还是远程方法
        - 保证各个组件之间的依赖界限
        - 注意异常处理：如果RPC Server宕掉很长时间了，该如何处理  
        如是你对RPC还不是很熟，就不要使用 RPC，如果你非要用，你可以用一个异步的管道来代替RPC，如blocking  
2. Callback queue<回调队列>  
    一般的，基于RabbitMQ的RPC是比较简单的，客户端发送请求，服务端响应请求。为了收到应答，客户端需要发送一个 **"callback"** 队列地址通过那个消息，如下：  
    ```python
    result = channel.queue_declare(exclusive=True)
    callback_queue = result.method.queue

    channel.basic_publish(exchange='',
                      routing_key='rpc_queue',
                      properties=pika.BasicProperties(
                            reply_to = callback_queue,
                            ),
                      body=request)
    ```
    >消息特性  
    AMQP协议预定义了消息的14个特性，除了以下几个，大多数的基本上用不着；
    - delivery_mode:消息持久化，value是2，其他值非持久化；
    - content_type:编码方式，如JSON encoding就设置成：application/json
    - reply_to: callback queue的名字
    - correlation_id: 用来关联RPC的响应和请求  
3. Correlation id  
    通过以上说明，建议每个RPC请求都创建一个回调队列callback queue；这非常低效，但是有另外一个更好的方法：一个客户端client只创建一个回调队列callback queue;  
    但这就有另外一个问题，我们收到队列的一个响应，但是在这个响应response里不清楚哪个响应对应哪个请求;此时，我们的correlation_id就派上了用场；我们通过它来标识每个请求；然后在callback queue收到response里，我们来对应这个值，这样就匹配上了每个请求的响应；如果我们看到了未知的correlation_id，我们就丢弃该消息，因为它不属于我们的请求；  
    你可能会问，为什么我们要忽略未知的请求，而不是直接报错？  
    这是因为server端有这种情况发生，尽管有异常的情况：如服务端将我们请求的结果返回给我们了，但是还没有应答这个请求；如果发生了这种情况，那么重启后的服务端会重新处理这个请求；这就是为什么我们在客户端要强制这样处理的原因

4. 总结  
如图：
![](/styles/images/rabbitmq-rpc01.png)
我们RPC工作流程其实是这样的：  
当客户端起动后，它会创建一个匿名独占回调队列  
对于一个RPC请求，客户端创建的消息包含两个属性： *reply_to* ，说白了就是回调队列名字；另一个： *correlation_id* ，说白了就是给每个request设置的惟一标识，或者说是它的名字；这个请求将会被发送到rpc_queue队列里；
RPC的worker在那个队列里等待消息，当请求出现时，它会执行任务并将结果返回给客户端，使用reply_to标识的队列  
客户端从回调队列等待接收数据，当消息出现时，它会检查correlation_id属性，如果跟请求的匹配，它会将该响应返回给应用；

### CODE
1. rpc_server.py  
    ```python
    #!/usr/bin/env python
    import pika

    connection = pika.BlockingConnection(pika.ConnectionParameters(host='localhost'))

    channel = connection.channel()

    channel.queue_declare(queue='rpc_queue')

    def fib(n):
        if n == 0:
            return 0
        elif n == 1:
            return 1
        else:
            return fib(n-1) + fib(n-2)

    def on_request(ch, method, props, body):
        n = int(body)

        print(" [.] fib(%s)" % n)
        response = fib(n)

        ch.basic_publish(exchange='',
                         routing_key=props.reply_to,
                         properties=pika.BasicProperties(correlation_id = \
                                                             props.correlation_id),
                         body=str(response))
        ch.basic_ack(delivery_tag = method.delivery_tag)

    channel.basic_qos(prefetch_count=1)
    channel.basic_consume(on_request, queue='rpc_queue')

    print(" [x] Awaiting RPC requests")
    channel.start_consuming()
    ```
2. rpc_client.py  
    ```python
    #!/usr/bin/env python
    import pika
    import uuid

    class FibonacciRpcClient(object):
        def __init__(self):
            self.connection = pika.BlockingConnection(pika.ConnectionParameters(host='localhost'))

            self.channel = self.connection.channel()

            result = self.channel.queue_declare(exclusive=True)
            self.callback_queue = result.method.queue

            self.channel.basic_consume(self.on_response, no_ack=True,
                                       queue=self.callback_queue)

        def on_response(self, ch, method, props, body):
            if self.corr_id == props.correlation_id:
                self.response = body

        def call(self, n):
            self.response = None
            self.corr_id = str(uuid.uuid4())
            self.channel.basic_publish(exchange='',
                                       routing_key='rpc_queue',
                                       properties=pika.BasicProperties(
                                             reply_to = self.callback_queue,
                                             correlation_id = self.corr_id,
                                             ),
                                       body=str(n))
            while self.response is None:
                self.connection.process_data_events()
            return int(self.response)

    fibonacci_rpc = FibonacciRpcClient()

    print(" [x] Requesting fib(30)")
    response = fibonacci_rpc.call(30)
    print(" [.] Got %r" % response)
    ```
3. bash执行  
    ```bash
    python rpc_server.py
    python rpc_client.py
    ```    

[rpc_server.py](/styles/rpc_server.py)  
[rpc_client.py](/styles/rpc_client.py)
