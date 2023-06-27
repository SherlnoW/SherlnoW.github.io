---
layout: post
title: MyRPC-Version3-使用Netty实现高性能网络通信
date: 2022-11-12
description: rpc-version3
tag:
- RPC
- Java
---

# 待解决问题

传统的BIO监听模式效率很低，可能会造成不必要的线程开销。

# 解决方案

网络传输可以改为NIO（异步阻塞IO），服务端通过一个线程处理多个请求，客户端发送的链接请求都会注册到多路复用器上，多路复用器轮询到链接有IO请求就进行处理。

Netty是一个基于NIO的高性能网络框架，用于开发高性能、高可靠的网络IO程序，知名的RPC框架（dubbo、grpc）都是使用netty底层进行通信。

# 背景知识

**NIO**

Java NIO实际上是多路复用IO，会有一个线程不断去轮询多个socket的状态，只有当socket真正有读写事件时，才真正调用实际的IO读写操作。这样，只有在真正有socket读写事件进行时，才会使用IO资源，大大减少了资源占用。

   <img src="E:\java learning\RPCByHand\imgs\3_NIO.png" alt="3_NIO" style="zoom: 67%;" />

   Java NIO的核心部件包括：通道（Channels）、缓冲区（Buffers）、选择器（Selectors），这些组件共同组成了多路复用机制。其中，选择器可以注册到多个通道上，监听各个通道上发生的事件，并根据事件情况决定调用线程处理。而缓冲区用于和通道进行数据交互，实质是一个可以读写的内存块。

   虽然Java的NIO在性能上优于BIO，但在高负载下如何高效、可靠地处理和调度I/O操作仍是一个问题，这就需要Netty出场了。

**Netty**

Netty是一个高性能的、异步通信的NIO框架，基于Java NIO的API实现，所有IO操作都是异步非阻塞的，它的高性能体现在以下方面： 
* IO 线程模型：同步非阻塞，用最少的资源做更多的事。
* 内存零拷贝：尽量减少不必要的内存拷贝，实现了更高效率的传输。
* 内存池设计：申请的内存可以重用，主要指直接内存。内部实现是用一颗二叉查找树管理内存分配情况。默认堆外内存，开启池化管理。
* 串形化处理读写：避免使用锁带来的性能开销。
* 高性能序列化协议：支持 protobuf 等高性能序列化协议。

Netty基于Reactor模式，通过多路复用器接收并处理用户请求，内部实现了两个线程池：boss线程池和work线程池。其中，boss线程池负责处理请求的accept事件，当接收到accept事件请求时，把对应的socket封装到一个NioSocketChannel中，并交给work线程池，work线程池负责请求的读写事件，在每个NioEventGroup处理业务时，会使用 pipeline（管道），通过pipeline可以获取到对应channel，管道中维护了很多的Handler用于数据的处理。

   <img src="E:\java learning\RPCByHand\imgs\3_Netty.png" alt="3_Netty" style="zoom:67%;" />

Netty框架的使用通过Maven引入对应的依赖实现，常用的类和初始化操作如下：

* **NioEventLoopGroup类:** 实际上是一个线程池，有可执行的Executor，同时继承了Iterable迭代器，里面包含了多个NioEventLoop。一个NioEventLoop可以处理多个Channel中的IO操作，但每个NioEventLoop只绑定一个线程，因此每一个NioEventLoop都绑定了一个Selector，负责决定当前的线程为哪些Channel提供服务。

* **ServerBootstrap类:** 通过链式操作初始化Netty服务器，负责监听端口的socket。ServerBoostrap用一个ServerSocketChannelFactory来实例化，可以选择NIO和BIO两种方式，但都需要两个线程池作为参数来进行初始化。ServerBootstrap.bind()方法可以绑定指定端口的socket连接，一个ServerBoosttap可以绑定多个端口，该方法会创建一个serverchannel，将当前的channel注册到eventloop上。

    ServerBoostrap常用的初始化设置如下：
  * group()：设置两个线程组
  * channel()：设置服务器的通道实现，这里传入NioServerSocketChannel负责new Socket
  * childHandler()：给Work Group的NioEventLoop对应的管道设置处理器

* **ChannelFuture类:** 由于Netty的IO操作全部是异步的，即任何IO操作会立即返回，但不保证这些IO操作在调用结束的时候已经完成，因此会返回一个ChannelFuture实例，负责提供关于IO操作的结果信息或状态信息。
  * channel()：返回ChannelFuture关联的Channel
  * channel().closeFuture().sync()：让线程进入wait状态直到监听到关闭事件，此时可以"优雅"地关闭通道和NettyServer

* **ChannelInitializer类:** 一种特殊的ChannelHandler，重写该类中的initChannel方法实现初始化一个Channel，用于在某个Channel注册到EventLoop后，对这个Channel执行一些初始化操作，初始化操作完成后会将子集从Pipeline中移除。
* **ChannelPipeline类:** 负责ChannelHandler的管理的事件拦截和调度。每当需要对Channel进行某种处理时，Pipeline负责依次调用每一个Handler进行处理。
* **Netty常用的解码器和编码器**
  * LengthFieldBasedFrameDecoder：基于长度的解码器，[长度\][消息体]，解决沾包问题
  * LengthFieldPrepender：编码器，计算当前待发送的消息长度，写入到规定字节中

**TCP沾包问题**

TCP沾包问题是指发送方发送的若干个数据包到接收方时沾成一个包。从接收缓冲区来看，后一个包数据的头紧接着前一个数据的尾。当TCP连接建立后，Client发送多个报文给Server，TCP协议虽然可以保证数据可靠性，但无法保证服务端接收到的包的数量和客户端一致，可能会出现少包或者多包的情况。

解决沾包问题通常有以下方法：
* 添加特殊符号，接收方通过这个特殊符号将受到的数据包拆分开
* 每次发送固定长度的数据包
* 在消息头中定义长度字段，来标识消息的总长度

# 具体实现

### common包

与上一版本相同，包括Blog类、User类，以及客户端请求RPCRequest和服务端回应RPCResponse。

### service包

与上一版本相同，包括服务端提供的UserService接口、BlogService接口以及它们对应的实现类。

### server包

服务端使用Netty进行改进。

* **RPCServer接口:** 实现服务端的类必需实现该接口。
* **NettyRPCServer类:** 实现RPCServer接口，作为服务端，负责监听和发送数据。按照Netty的工作原理定义了两个NioEventLoopGroup：boosGroup和workGroup，前者负责处理连接请求，后者负责实际业务的处理。之后通过ServerBootstrap完成Netty服务端的初始化。接着开启同步阻塞，并进行死循环监听。
* **NettyServerInitializer类:** 负责进行初始化，主要负责序列化的编码和解码，同时需要解决Netty的沾包问题。

    Netty基于流并通过Channel和Buffer实现消息传递，因此Object也需要转换成Channel和Buffer来传递。这里使用Netty提供默认的编码解码工具：ObejctEncoder()和ObjectDecoder()，这是一组Handler。而如果要实现自定义的编码解码，也应该在Handler中实现。 

    使用LengthFieldBasedFrameDecoder、LengthFieldPrepender实现解码器和编码器可以实现基于长度的数据传输，解决沾包问题。

* **NettyRPCServerHandler类:** 负责针对不同的请求数据类型调用handler进行处理。自定义一个Handler需要继承Netty规定好的HandlerAdapter，这里使用SimpleChannelInboundHandler，传入泛型为RPCRequest。
    * channelRead0()：重写方法，负责读取客户端发送的消息，并通过反射调用服务端实现类，并将结果response写入到缓存，并刷新。
    * exceptionCaught()：重写方法，负责处理异常和关闭通道。
    * getResponse()：类似于WordThread中的getResponse()，通过反射调用对应的方法。

* **TestServer类:** 服务端启动类，这里改为Netty版本。

* ServiceProvider类: 与上一个版本相同，使用Map保存接口名和实现类。

### clien包

这里按照开放封闭原则、单一职责原则对客户端进行重构（与服务端类似），之后使用Netty对客户端进行改进。

* **RPCClient接口:** 将客户端抽象为一个接口，后面的客户端启动类实现这个接口即可。其中只包含一个sendRequest()方法，负责发送request，并返回response。
* **RPCClientProxy类:** 客户端代理，将动态代理封装成request对象。与上个版本不同的是，这里通过RPCClient发送request，只需要在参数中放入request，而host和port则通过真正的客户端传入。
* **SimpleRPCClient类:** 类似于上个版本的IOClient类，负责底层通信。

接下来使用Netty对客户端进行改进。

* **NettyRPCClient类:** 客户端实现类，实现了RPCClient接口。这里传入的Bootstrap类似于服务端中的ServerBootstrap。之后进行客户端的初始化（与服务端类似）。

  在重写sendRequest()方法时，需要注意，由于Netty都是异步传输，发送request会立刻返回，但得到的返回并不是相对应的response，因此，这里在建立通信连接并发送数据后，通过给channel设计别名，获取特定名字的channel中的内容。

* **NettyClientInitializer类:** 采用与服务端相同的编码、解码方式。

* **NettyClientHandler类:** 自定义的客户端Handler，在重写的channelRead0()方法中，通过AttributeKey给channel设计别名，让sendRequest读取对应的response。AttributeMap是一个数组+链表结构的线程安全的Map，是绑定在Channl或ChannelHandlerContext上的一个附件，前者的AttributeMap是共享的，而后者的AttributeMap是独有的。可以通过AttributeKey（key）找到对应的Attribute（value）。

* **TestClient类:** 客户端启动类，这里改用Netty版本。

# 运行结果

1. 基于Netty的服务端启动后，接收到客户端发送的request，并返回response

   ![3_服务端启动](E:\java learning\RPCByHand\imgs\3_服务端启动.png)

2. 客户端接收到服务端发送的response

   ![3_客户端接收结果](E:\java learning\RPCByHand\imgs\3_客户端接收结果.png)

# 不足之处

目前为止使用的序列化方式都是Java自带的序列化，但这种方式有很多缺点：无法跨语言、序列后的码流太大、性能低等。