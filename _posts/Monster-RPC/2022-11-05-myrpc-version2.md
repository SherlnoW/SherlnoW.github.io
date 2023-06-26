---
layout: post
title: MyRPC-Version2-提供多个服务接口
date: 2022-11-05
description: rpc-version2
tag:
- RPC
- Java
---

# 待解决问题

如果服务端需要提供多个服务的接口，应该如何改进？

# 解决方案

可以使用一个Map保存服务接口（key）和其对应的实现类（value）。
```java
UserService userService = new UserServiceImpl();
BlogService blogService = new BlogServiceImpl();
Map<String, Object>.put(".userService", userService);
Map<String, Object>.put(".blogService", blogService);
//可以通过request调用对应的服务
Object service = map.get(request.getInterfaceName());
```

# 背景知识

**代码解耦**

耦合：指两个类之间联系的紧密程度，类之间存在直接关系称为强耦合，类之间只有间接关系则称为弱耦合。

解耦：降低耦合度，即将强耦合变为弱耦合的过程。

对象之间耦合越高，维护成本越高，因此需要把依赖关系降到最低，而不会导致牵一发而动全身的情况发生。

**线程池**

线程过多会带来调度开销，进而影响缓存局部性和整体性能，而线程池维护着多个线程，对线程统一管理，线程池里面存放着很多可以复用的线程。

# 具体实现

### service包

相较于上一版，增加一个 BlogService 服务接口。

* **BlogService接口:** 通过某个id查询Blog信息
* **BlogServiceImpl类:** 实现 BlogService接口的功能：通过id查询Blog信息。

### common包

相较于上一版，增加 Blog类。

* **Blog类:** 包括 id、userId、title三个属性。

### server包

服务端代码进行重构。

* **ServiceProvider类:** 本质上是一个HashMap，用于存放服务端调用的接口名（key）和对应的实现类（value）。服务端启动后，暴露相关的实现类，根据request中的接口名（interface）调用相应的实现类。
    * provideServiceInterface(Object service) 方法：通过反射获得传入service的接口名。并将接口名和对应实现类存入 Map<String, Object\> interfaceProvider 字段中。

* **RPCServer接口:** 把RPCSever抽象成一个接口，服务端实现这个接口即可，遵循开放封闭原则。
    > 开放封闭原则：软件实体应当对扩展性开放，而对修改原代码关闭。

* **SimpleRPCServer类:** 实现了 RPCServer接口。使用Java原始的BIO监听模式，每当监听到一个request后，就创建一个线程进行处理。
* **ThreadPoolRPCServer类:** 实现了 RPCServer接口。通过线程池实现服务端，同样采用Java原始的BIO监听模式。每当监听到一个request后，就通过线程池里的线程进行处理。
* **WorkThread类:** 将对监听到的客户端的request进行解析、执行相应的服务以及返回response给客户端的功能从服务端代码中分离开来，遵循单一职责原则。
  > 单一职责原则：一个对象应当只包含单一的职责，并且该职责被完整封装在一个类中。
* **TestService类:** 服务端启动类，首先暴露相关服务的实现类，之后通过 ServiceProvider类将接口名和对应实现类存入 HashMap 中，启动服务端，开始监听。

### client包
* **RPCClient类:** 客户端启动类，负责调用服务端的方法。除了原有的查询User和插入User外，增加一个查询Blog的请求。

# 运行结果

1. 服务端启动后，接收到客户端发送来的request，并发送response
    <figure>
        <img src="https://s1.ax1x.com/2023/06/26/pCUBPbt.png" alt="服务端启动" >
        <figcaption>Fig 1. 服务端启动.</figcaption>
    </figure>

2. 客户端接收到服务端发送的response
    <figure>    
        <img src="https://s1.ax1x.com/2023/06/26/pCUBVPS.png" alt="客户端启动" >
        <figcaption>Fig 2. 客户端启动.</figcaption>
    </figure>

# 不足之处

传统的BIO与线程池的网络传输性能低。