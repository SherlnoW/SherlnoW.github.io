---
layout: post
title: MyRPC-Version1
date: 2022-10-29
description: version1
tag:
- RPC
- Java
---

## Version1 —— 支持多种远程方法调用和返回值类型

### 待解决问题

1. 增加客户端可以请求的远程服务
2. 增加服务端返回值的类型

### 解决方案

1. 增加客户端可以请求的远程服务

通过定义一个通用的请求对象（Request），服务端通过客户端发送的需要调用的Service接口名、方法名、参数、参数类型，根据反射调用相应的方法。

2. 增加服务端返回值的类型

通过定义一个通用的响应对象（Response），将传输对象抽象成Object，而不是某一特定类型的数据。

### 基础知识

1. 反射

反射赋予了我们在运行时分析类以及执行类中方法的能力，通过反射可以获取任意一个类的所有属性和方法，并进行调用。

2. 动态代理

代理模式：通过创建代理类（proxy）的方式来访问服务，代理类通常会持有一个委托类对象，代理类不会自己实现真正的服务，而是通过调用委托类对象的相关方法提供服务。

动态代理：代理类在运行时动态生成，以方便对代理类所有函数进行统一管理。

Java动态代理机制中实现动态代理的核心为 InvocationHandler（接口）和 Proxy（类）：

* InvocationHandler接口

每一个动态代理类的调用处理程序都必须实现 InvocationHandler接口，并且每个代理类的实例都关联到了实现该接口的动态代理类调用处理程序中，当通过动态代理对象调用一个方法时，这个方法的调用就会被转发，去调用实现 InvocationHandler接口类 的 invoke()方法。

* Proxy类

用于动态生成代理类：Proxy.newProxyInstance(loader, interfaces, h) —— 传入类加载器、一组接口及调用处理器生成动态代理类实例（会调用该实例中的invoke()方法执行）。

### 具体实现

#### common包

网络在传输过程中不再使用User对象，而是使用request和response格式的数据进行传输，客户端和服务端只对这些格式的数据进行封装和解析。

* RPCRequest类

定义了一个通用的Request对象（消息格式） ，负责将客户端发送的请求扩展为所调用的Service的接口名、方法名、参数值以及参数类型，以便服务端通过反射调用相应的方法。

* RPCResponse

定义了一个通用的Response对象（消息格式），负责将服务端传回的响应抽象为一个Object，同时引入状态码，表示服务是否调用成功，若没有调用成功则传回错误信息。

* User类

用于操作的对象。

#### service包

* UserService接口

客户端通过此接口对服务端的实现类进行调用。这里定义了两个抽象方法：getUserByUserId、insertUserId，分别负责使用UserId查询User和插入一条User。

* UserServiceImpl类

负责实现UserService接口中的具体功能，这里增加了插入一条User的功能。

#### server包

* RPCServer类

服务端，负责解析客户端发来的request以及封装回复给客户端的response对象。

与version0相比，这里在读取到客户端传来的request后，通过反射调用相应的方法，然后再进行封装，写入response对象，发送给客户端。

```java
Method method = userService.getClass()
.getMethod(request.getMethodName(),request.getParamsTypes());
Object invoke = method.invoke(userService, request.getParams());
```

#### client包

客户端根据不同的Service进行动态代理：将不同的Service方法封装成统一的Request对象格式，并建立与服务端的通信。

* IOClient类

负责底层与服务端的通信，当客户端发起一次调用请求后，Socket建立起与服务端的连接，然后发送由上层封装好的request请求，再得到服务端发送回来的response响应。

* ClientProxy类

动态代理封装request对象。

不同的service需要进行不同的封装，而客户端只知道Service接口，这就需要一层动态代理根据反射封装不同的Service。

* RPCClient类

客户端，负责调用不同的方法，这里增加了插入一个User的服务调用。

### 运行结果

1. 启动服务端，监听8080端口

<figure>
<img src="https://s1.ax1x.com/2023/06/26/pCU0zgH.png" alt="服务端启动" >
<figcaption>Fig 1. 服务端启动.</figcaption>
</figure>


2. 启动客户端，

<figure>
<img src="https://s1.ax1x.com/2023/06/26/pCUB9KA.png" alt="服务端提供服务" >
<figcaption>Fig 2. 服务端提供服务.</figcaption>
</figure>

<figure>
<img src="https://s1.ax1x.com/2023/06/26/pCUBkUf.png" alt="客户端接受结果" >
<figcaption>Fig 3. 客户端接受结果.</figcaption>
</figure>

### 不足之处

1. 这里只有一个UserService服务，如果有多个服务如何进行注册？
2. 服务端采用BIO方式监听Socket，性能太低。
3. 服务端的功能过于复杂，既要进行监听，又要对request进行处理，耦合度过高。

