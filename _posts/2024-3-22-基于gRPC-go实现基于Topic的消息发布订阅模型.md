---
layout: post
published: true
title: 2024-3-22-基于gRPC-go 实现基于Topic的发布订阅模式
categories: [gRPC,go]
tags: [gRPC,go]
---
* content
{:toc}


# 背景

由于要在GoCN社区分享相关主题，因此整理了这篇文章。您可以在[这里](https://github.com/zackzhangkai/grpc-pubsub-example/blob/main/docs/%E5%89%AF%E6%9C%AC2_3_1_gocn-%E5%BC%A0%E5%87%AF-%E5%9F%BA%E4%BA%8EgRPC%20go%E5%AE%9E%E7%8E%B0%E6%B6%88%E6%81%AF%E5%8F%91%E5%B8%83%E8%AE%A2%E9%98%85.pdf)查看原文。

# 什么是gRPC

gRPC是Google开源的基于HTTP/2和ProtoBuf的通用RPC框架，它可用于快速开发可复用的分布式应用。gRPC支持多种语言，如Go、Java、Python、C++等，并且可以运行在多种环境中，如Windows、Linux、Mac等。

您可以在这里获取到更多关于gRPC的信息：[gRPC 官方文档](https://grpc.io/)

gRPC是七层协议，它基于HTTP/2协议，使用Protobuf作为序列化格式。适合视频直播、游戏、物联网、微服务等数据量大的场景。

# 什么是ProtoBuf

Protobuf是Google开源的跨语言、跨平台的结构化数据序列化库，它可用于通信协议、数据存储等场景。

示例：

```protobuf
// 定义一个GetPersonService的服务，这个服务里有一个GetPerson的RPC方法
service GetPersonService {
  rpc GetPerson (GetPersonRequest) returns (Person) {}
}

// 定义一个Response Person的消息类型
message Person {
  required string name = 1;
  required int32 id = 2;
  optional string email = 3;
  }

// 定义一个Request GetPersonRequest的消息类型
message GetPersonRequest {
  required string name = 1;
}
```

通过这个例子，我们可以看到Protobuf是如何定义消息类型的。


# gRPC四种连接模式

gRPC 支持四种不同的连接模式或通信模型，每种模式都适应不同的场景需求，下面对这四种模式进行详细描述：

## 一元（Unary）RPC 在一元 RPC 中，客户端发送一个请求给服务器，然后服务器处理请求并返回一个响应。这是最传统的请求-响应模式，类似于 HTTP 的 GET 或 POST 请求。

```Protobuf
// 一元 RPC 示例
rpc SayHello (HelloRequest) returns (HelloReply) {}
```

在此模式中，SayHello 方法接受一个 HelloRequest 参数，并返回一个 HelloReply 结果。

## 服务器端流式（Server-Side Streaming）RPC 在服务器端流式 RPC 中，客户端发送一个请求给服务器，但服务器会连续发送多个响应返回给客户端。这种模式适用于需要持续推送数据的情况，比如实时事件更新或逐条检索数据库记录。

```Protobuf
// 服务器端流式 RPC 示例
rpc ServerStreamingMethod (StreamingRequest) returns (stream StreamingResponse) {}
```

在这个例子中，当客户端调用 ServerStreamingMethod 并发送一个 StreamingRequest 后，服务器可以发送一系列的 StreamingResponse 数据包。

## 客户端流式（Client-Side Streaming）RPC 在客户端流式 RPC 中，客户端可以连续发送多个请求到服务器，而服务器只返回一个响应。这在需要批量处理或聚合多个请求时很有用。

```Protobuf
// 客户端流式 RPC 示例
rpc ClientStreamingMethod (stream ClientRequest) returns (SingleResponse) {}
```
在上述示例中，客户端可以分多次发送多个 ClientRequest 给服务器，最后服务器一次性返回一个 SingleResponse。

## 双向流式（Bidirectional Streaming）RPC 双向流式 RPC 允许客户端和服务端同时发送多个请求和响应。这是一种全双工通信方式，两边都可以连续发送消息。

```Protobuf
// 双向流式 RPC 示例
rpc BidirectionalStreamingMethod (stream BidirectionalRequest) returns (stream BidirectionalResponse) {}
```
在双向流式 RPC 中，客户端和服务器可以同时进行多次请求和响应的交换，例如在文件传输、聊天应用或协同编辑场景中可能会使用这种模式。

由于gRPC是一个rpc调用的框架，换言之：使用api调用不友好，典型比如通过浏览器访问时，无法通过grpc协议实现。

因此社区在探索这方面的解决方案，比如grpc-web、grpc-gateway等。

grpc-web是通过envoy做中间代理，将grpc协议转换为http协议，从而实现浏览器访问grpc服务。
grpc-gateway则是通过在protobuf中定义http接口，实现浏览器访问grpc服务。

# 模型实现实现



# 相关

[gRPC 官方文档](https://github.com/grpc/grpc-go/tree/master/examples/route_guide)
[grpc-web](https://github.com/grpc/grpc-web/blob/master/net/grpc/gateway/examples/helloworld/README.md)
[grpc-gateway](https://github.com/grpc-ecosystem/grpc-gateway/blob/main/README.md)

