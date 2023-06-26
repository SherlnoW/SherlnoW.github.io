---
layout: post
title: MyRPC-Version0-一个最简单的RPC调用
date: 2022-10-22
description: rpc-version0
tag:
- RPC
- Java
---

# 待解决的问题

1. RPC的基本过程是什么？
2. 怎样完成一个RPC？

# 解决方案

**RPC的基本过程是什么？**

客户端调用服务端的一个方法，服务端通过执行本地方法得到返回值，并将返回值传递回客户端。

**怎样完成一个RPC？**

以下是假设的服务背景：

服务端：拥有一张User表

客户端：通过传递想要查询的id给服务端，并得到服务端返回的User对象

# 基础知识

### Socket通信

基于TCP/IP网络层的一种传送方式，属于传输层
* 首先，服务端初始化ServerSocket，然后对指定的端口进行绑定，接着对端口进行监听，
* 通过调用 `accept()` 进行阻塞，此时，若客户端有一个 socket 连接到服务端，那么服务端
* 通过监听和 `accept()` 可以与客户端进行连接。

### lombok

一个在Java开发过程中用注解的方式，简化了 JavaBean 的编写，避免了冗余和样板式代码而出现的插件。

常用注解如下：
* @Builder：作用于类，将其变成建造者模式，一步步地创建一个对象（屏蔽具体细节，但可以精细控制）
* @Data：在类上标注，自动生成相关的 `get()、set()、equals()、hashCode()、toString()` 等方法
* @NoArgsConstructor：在类上标注，提供类的无参构造
* @AllArgsConstructor：在类上标注，提供类的全参构造

### Serializable接口

用于实现Java类的序列化操作而提供的一个语义级别的接口。

实现了Serializable接口的类可以被ObjectOutPutStream转换为字节流，也可通过ObjectInputStream再将其解析为对象。

### BIO通信

同步阻塞I/O模式，数据的读取写入必须阻塞在一个线程内等待其完成。

最容易实现的IO工作方式，应用程序向操作系统请求网络IO操作，这时应用程序会一直等待；另一方面，操作系统收到请求后，也会等待，直到网络上有数据传到监听端口；操作系统在收集数据后，会把数据发送给应用程序；最后应用程序受到数据，并解除等待状态。

# 具体实现

### common包

用来存放客户端查询、服务端提供的数据（对象）

* **User类:** User对象-客户端和服务端都已知，其中客户端想要得到这个pojo对象数据，服务端需要对此对象进行操作。（这里User对象属性包括id、userName、sex）

### service包

用来存放服务端所提供的服务接口

* **UserService接口:** 客户端通过调用此接口来调用服务端的实现类，以达到通过id查询User对象。
* **UserServiceImpl类:** 实现UserService接口中的方法，具体实现通过id查询User对象的功能。

### server包

服务端，用来接收客户端的请求并提供相应的服务

* **RPCServer类:** 以BIO的方式监听Socket，如果有数据，则调用相应的实现类提供服务，并返回结果给客户端

### client包

客户端，包含发送给服务端的请求以及得到的返回对象

* **RPCClient类:** 建立Socket连接，向服务端传输id，并得到返回的User对象。

# 结果

1. 启动服务端，监听8080端口
   <figure>
   <img src="https://s1.ax1x.com/2023/06/26/pCUBSvd.png" alt="服务端启动" >
   <figcaption>Fig 1. 服务端启动.</figcaption>
   </figure>

2. 启动客户端，传输id查询User对象，服务端调用相应实现类提供服务，并返回结果给客户端
   <figure>
   <img src="https://s1.ax1x.com/2023/06/26/pCUBCDI.png" alt="客户端启动服务端提供服务" >
   <figcaption>Fig 2. 客户端启动服务端提供服务.</figcaption>
   </figure>

# 不足之处

1. 只能调用服务端Service唯一确定的方法，如果有多个方法需要调用呢？（Request需要抽象）
2. 返回值只支持User对象，如果返回值是其他类型呢？（Response需要抽象）
3. 客户端的host、port以及所调用的方法都进行了指定，还不够通用。