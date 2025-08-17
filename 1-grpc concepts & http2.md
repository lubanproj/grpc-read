> 本文原发表于：https://diu.life/lessons/grpc-read/grpc-concepts-http2/
> 最新版本请访问原文链接

### 写在前面

#### grpc 介绍

grpc 是 google 开源的一款高性能的 rpc 框架。github 上介绍如下：

	gRPC is a modern, open source, high-performance remote procedure call (RPC) framework that can run anywhere

市面上的 rpc 框架数不胜数，包括 alibaba dubbo 和微博的 motan 等。grpc 能够在众多的框架中脱颖而出是跟其高性能是密切相关的。

#### CONCEPTS

阅读 grpc 源码之前，我们不妨先了解一些 concepts，github 上也有一些 concepts 介绍

https://github.com/grpc/grpc/blob/master/CONCEPTS.md

##### 1.接口设计

	Developers using gRPC start with a language agnostic description of an RPC service (a collection of methods). From this description, gRPC will generate client and server side interfaces in any of the supported languages. The server implements the service interface, which can be remotely invoked by the client interface.
	
	By default, gRPC uses Protocol Buffers as the Interface Definition Language (IDL) for describing both the service interface and the structure of the payload messages. It is possible to use other alternatives if desired.

对一个远程服务 service 的调用，grpc 约定 client 和 server 首先需要约定好 service 的结构。包括一系列方法的组合，每个方法定义、参数、返回体等。对这个结构的描述，grpc 默认是用 protocol buffer 去实现的。

##### 2. Streaming

streaming 在 http/1.x 已经出现了，http2 实现了 streaming 的多路复用。grpc 是基于 http2 实现的。所以 grpc 也实现了 streaming 的多路复用，所以 grpc 的请求有四种模式：Simple RPC、Client-side streaming RPC、Server-side streaming RPC、Bidirectional streaming RPC 。也就是说同时支持单边流和双向流

##### 3. Protocol

grpc 的协议层是基于 http2 设计的，所以你如果想了解 grpc 的话，可以先深入了解 http2

##### 4. Flow Control

grpc 的协议支持流量控制，这里也是采用了 http2 的 flow control 机制。


通过上面的介绍可以看到，grpc 的高性能很大程度上依赖了 http2 的能力，所以要了解 grpc 之前，我们需要先了解一下 http 2 的特性。



#### http2 特性


1. 二进制协议
	
	众所周知，二进制协议比文本形式的协议，发送的数据量肯定是更小，传输效率更高的。所以 http2 比 http/1.x 更高效，因为二进制是不可读的，所以会损失部分可读性。
	
2. 多路复用的流
   
   http/1.x 一个 stream 是需要一个 tcp 连接的，其实从性能上来说是比较浪费的。http2 可以复用 tcp 连接，实现一个 tcp 连接可以处理多个 stream，同时可以为每一个 stream 设置优先级，可以用来告诉对端哪个流更重要。当资源有限的时候，服务器会根据优先级来选择应该先发送哪些流
   
3. 头部压缩

   由于 http 协议是一个无状态的协议，导致了很多相同或者类似的 http 请求重复发送时，带的头部信息很多时候是完全一样的。http2 对头部进行压缩，可以减少数据包大小，提高协议性能
 
4. 请求 reset

   在 http/1.x 中，当一个含有确切值的 Content-Length 的 http 消息发出之后，需要断开 tcp 连接才能中断。这样会导致需要通过三次握手来重新建立一个新的 tcp 连接，代价是比较大的。在 http2 里面，我们可以通过发送 RST_STREAM 帧来终止当前消息发送，从而避免了浪费带宽和中断已有的连接。

5. 服务器推送

	如果一个 client 请求资源 A，而 server 知道 client 可能也会需要资源 B， 所以在 client 发起请求前，server 提前将 B 推送给 A 缓存起来，从而可以缩短资源 A 这个请求的响应时间。

6. flow control
	
	在 http2 中，每个 http stream 都有自己公示的流量窗口，对于每个 stream 来说，client 和 server 都必须互相告诉对方自己能够处理的窗口大小，stream 中的数据帧大小不得超过能处理的窗口值。



这是题主简单了解，并总结了一下，对 http2，网上有很多文章介绍的非常详细，大家可以自行深入研究下。
