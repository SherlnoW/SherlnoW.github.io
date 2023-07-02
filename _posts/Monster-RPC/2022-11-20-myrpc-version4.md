---
layout: post
title: MyRPC-V4-实现自己的传输协议
date: 2022-11-20
description: rpc-version4
tag:
- RPC
- Java
---

# 待解决问题

目前使用的序列化方式是JDK自带的序列化，只需实现 java.io.Serializable 接口即可。但这种方式序列化性能和跨语言等方面有缺陷。

# 解决方案

除了Java原生序列化，还有许多其他常用的序列化协议，如kryo、hessian、protostuff等。此外，还可以自定义传输格式和编解码，实现自己的序列化方式。

以下是自定义的编码格式：依次读取消息类型、序列化方式，同时加上消息长度以防沾包，根据长度读取数据。

| 消息类型 | 序列化方式 | 消息长度 | 序列化字节数组 |
| :------: | :--------: | :------: | :------------: |
|  2Byte   |   2Byte    |  4Byte   |       -        |

# 背景知识

**序列化和反序列化**

序列化：将数据结构或对象转换成二进制字节流的过程

反序列化：将在序列化过程中所生成的二进制字节流转换成数据结构或者对象的过程

在Java中，序列化的对象就是实例化后的类，主要目的是通过网络传输对象。序列化协议在OSI七层协议中对应表示层，在TCP/IP协议中属于应用层的一部分。

**常见的序列化协议**

* JDK自带的序列化方式: 只需要实现 java.io.Serializable 接口即可，但由于性能、跨语言和安全性等方面的缺陷，几乎不会直接使用。
* Kryo: 一个高性能的序列化/反序列化工具，由于其变长存储特性并使用了字节码生成机制，拥有较高的运行速度和较小的字节码体积。目前已经是一种非常成熟的序列化实现了。
* Protobuf: 出自Google，性能较为优秀，支持多种语言，且跨平台。但使用中需要自定义IDL文件和生成对应的序列化代码，繁琐且不灵活，但解决了序列化漏洞的风险。
* hessian: 一个轻量级、自定义描述的二进制RPC协议，dubbo RPC默认采用hessian2序列化（进行了修改，大体结构相同）。
* Json序列化方式: 将对象序列化为JsonString进行网络传输，常见的有Jackson、Gson、FastJson等。

3. **自定义序列化协议**

通俗来说，就是自定义一套数据传输的规则。在传输过程中，RPC并不会把请求的所有二进制数据一下全部发送，而是拆分成多个数据包，也可能会合并其他请求的数据包（要求是同一个TCP连接上的请求）。这个过程有许多的细节，因此，在自定义传输协议时，需要包含以下几个必要的元素：
* 消息长度

对发送消息的长度进行标识。服务端或客户端在接收消息时，通过读取固定位置的数据得知本次发送的数据长度，从而识别出此次需要读的完整数据，防止沾包。

* 序列化方式

RPC可能支持多种序列化方式，因此，协议里需要含有序列化方式的标识，以便得到数据后能够正常反序列化。

* 魔数

对于不同类型的文件，需要在文件开头用一个固定的字符或者数字标识，将其称为魔数。

除此之外，一个好的序列化协议还应该包括协议头长度、协议版本等，以保证协议的可扩展性。

# 具体实现

### common包

* **RPCResponse类:** 与上一个版本相比，增加了 Class<?> dataType，以便在使用其它序列化方式时可以得到数据类型。

* RPCRequest类、Blog类、User类: 与上一个版本相同。

### service包

与上一个版本相同，包括服务端提供的UserService接口、BlogService接口以及对应的实现类，同时包括使用Map保存接口名和实现类的ServiceProvider类。

### codec包

包含自定义的接编码器以及所支持的各种不同的序列化器。

* **Serializer接口:** 序列化器的接口，凡是充当序列化器的类必须实现这个接口，其中包含的方法如下：

    * `byte[] serialize(Object obj);`: 负责将对象序列化成字节数组。
    * `Object deserialize(byte[] bytes, int messageType);`: 负责将字节数组反序列化为对象，其中Java原生的序列化方式不需要messageType（序列化字符数组里包含类信息），而其他方式必须指定消息格式。
    * `int getType();`: 返回所使用的序列化器，其中：0代表Java原生序列化方式；1代表json序列化方式；2代表kryo序列化方式。
    * `static Serializer getSerializerByCode(int code){}`: 根据传入的序号取出序列化器，目前支持的有Java自带序列化方式、json序列化方式、kryo序列化方式。

* **ObjectSerializer类**

  Java原生的序列化器，实现了Serializer接口。通过Java IO实现对象和字节数组之间的相互转换，并在getType()方法中返回0，代表使用的是Java原生序列器。实际的数据流向为：ObjectOutputStream -> ByteArrayOutputStream -> ByteArrayInputStream -> ByteArrayInputStream

* **JsonSerializer类**

  Json序列化器，实现了Serializer接口。这里通过使用阿里的FastJson实现对象和字符串之间的相互转化，但由于对象转化为字符串会丢失对象的类信息，因此需要在deserialize中通过messageType将jsonObject转换为对象。同时在getType()方法中返回1，代表使用的是json序列化方式。

* **KryoSerializer类**

  Kryo序列化器，实现了Serializer接口。由于Kryo是线程不安全的，因此需要借助ThreadLocal保证其线程安全，每个线程都使用独立的kryo，同时对循环依赖检测、循环引用检测进行了设置，并设定了实例化器。

* **MessageType枚举类**

  消息类型，包括REQUEST(0)和RESPONSE(1)。

* **MyEncode类**

  自定义编码器，按照自定义的消息格式依次写入数据：消息类型（request或response）、序列化方式（Java原生序列化器或Json序列化器）、消息长度、序列化字节数组。

* **MyDecode类**

  自定义解码器，按照自定义的消息格式依次解码得到数据：消息类型、序列化方式、消息长度、序列化节字数组。

### server包

* **NettyServerInitializer类**

  与上一版本不同的是，pipline这里加入自定义的编解码器，同时在编码器中选择序列化方式。

* RPCServer接口、NettyRPCServer类、NettyRPCServerHandler类、TestServer类

  与上一版本相同。

### client包

* **NettyClientInitializer类**

  与上一版本不同的是，pipline这里加入自定义的编解码器，同时在编码器中选择序列化方式。

* RPCClient接口、RPCClientProxy类、NettyRPCClient类、NettyClientHandler类、TestClient类

  与上一版本相同。

# 运行结果

1. 服务端启动后，接收到客户端发送的request，并返回response，可以看到服务端使用的是Json序列化器
<figure>
    <img src="https://s1.ax1x.com/2023/06/26/pCUsMqA.png" alt="4_服务端启动" style="zoom: 67%;" >
    <figcaption>Fig 1. 服务端启动.</figcaption>
</figure>

2. 客户端发送request后接收到服务端发回的response，客户端使用是Kryo序列化器
<figure>
    <img src="https://s1.ax1x.com/2023/06/26/pCUslVI.png" alt="4_客户端接收结果" style="zoom: 67%;" >
    <figcaption>Fig 1. 客户端接收结果.</figcaption>
</figure>

# 不足之处

在通信过程中，每一个客户端都需要知道对应服务的ip和端口号，并且当服务端挂了或更换地址后，再次通信的话很麻烦，扩展性不强。