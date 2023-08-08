---
layout: post
title: 【项目】MyRPC-V7-使用Nacos作为注册中心
date: 2022-12-17
description: rpc-version7
tag:
- RPC
- Java
---

# 待解决问题

从CAP角度考虑，zookeeper是一个CP系统，具有强一致性（集群leader挂了会重新选举，此时暂停对外服务），但注册中心绝不允许因为注册中心出现问题而导致服务间的调用出问题，应该是个AP系统。

# 解决方案

将Zookeeper更换为Nacos作为注册中心。

# 背景知识

**Nacos**

Nacos是阿里推出的一款开源工具，用于实现分布式系统的服务发现和配置管理。Nacos依赖Mysql数据库做数据存储，当有数据更新时，直接更新数据库的数据，然后将数据更新的信息异步广播给Nacos集群中所有服务节点数据变更，再由Nacos服务节点更新本地缓存，然后通知客户端节点数据变化。

Nacos支持两种方式的注册中心：持久化和非持久化存储服务信息。
* 持久化: 使用Raft协议选举master节点，与zookeeper一样采用过半机制将数据存储在leader节点上
* 非持久化: 直接存储在Nacos服务节点的内存中，并且服务节点间采用去中心化的思想，服务节点采用hash分片存储注册信息。

Nacos同时实现了CP和AP两种数据的一致性策略，AP模式下服务以临时实例注册，CP模式下服务以永久实例注册。

**注册中心的对比**
<figure>
    <img src="https://s1.ax1x.com/2023/07/02/pCDMg9s.png" alt="7_注册中心对比" style="zoom: 67%;" >
    <figcaption>Fig 1. 注册中心对比.</figcaption>
</figure>

# 具体实现

### 安装Nacos

首先安装Nacos，并修改`/conf`目录下的`application.properties`。为了简单，这里将启动模式修改为单机模式，即`nacos.standalone = true`，启动`/bin`目录下的`startup.cmd`，可以看到Nacos正常启动，默认端口为8848。
    
<figure>
    <img src="https://s1.ax1x.com/2023/07/02/pCDMBB8.png" alt="7_Nacos启动" style="zoom: 67%;" >
    <figcaption>Fig 2. Nacos启动.</figcaption>
</figure>

在浏览器中输入`http://localhost:8848/nacos/`进入Nacos的控制台，账户和密码均为`nacos`，可以看到，此时还没有服务注册进来。

<figure>
    <img src="https://s1.ax1x.com/2023/07/02/pCDMyNQ.png" alt="7_Nacos控制台" style="zoom: 67%;" >
    <figcaption>Fig 3. Nacos控制台.</figcaption>
</figure>

### register包

* **NacosServiceRegister类:** Nacos作为注册中心的实现类，实现了ServiceRegister接口。通过NamingFactory连接Nacos。NamingService通过registerInstance()方法传入服务名、IP地址和端口号，像Nacos注册服务；通过getAllInstances()方法传入服务名获得提供该服务的所有提供者列表，并通过负载均衡算法选择其中一个。
* ServiceRegister接口 与上一个版本相同。

### loadbalance包

由于Nacos Client在进行服务注册时，通过Instance对象来携带实例的基本信息，所以这里需要对负载均衡代码进行修改。

* **loadBalance接口、RandomLoadBalance类、RoundLoadBalance类:** 将String对象改为Instance对象。

### service包

* **ServiceProvider类:** 将serviceRegister修改为NacosServiceRegister类。
* UserService接口、UserServiceImpl类、BlogService接口、BolgServiceImpl类  与前一个版本相同。

### client包

* **NettyRPCClient类:** 将serviceRegister修改为NacosServiceRegister类。
* TestClient类、RPCClient接口、RPCClientProxy类、NettyClientInitializer类、NettyClientHandler类 与前一个版本相同。

### codec包、common包、server包

遇前一个版本相同。

# 运行结果

1. 在保持Nacos启动后，启动服务端，与Nacos连接成功。
    <figure>
        <img src="https://s1.ax1x.com/2023/07/02/pCDM23n.png" alt="7_服务端启动" style="zoom: 67%;" >
        <figcaption>Fig 4. 服务端启动.</figcaption>
    </figure>

2. 查看Nacos控制台，可以看到两个服务已经注册到了Nacos中，通过详情可以看到该服务的IP地址、端口号等信息。
    <figure>
        <img src="https://s1.ax1x.com/2023/07/02/pCDM6hj.png" alt="7_Nacos服务列表" style="zoom: 67%;" >
        <figcaption>Fig 5. Nacos服务列表.</figcaption>
    </figure>

    <figure>
        <img src="https://s1.ax1x.com/2023/07/02/pCDMsAg.png" alt="7_Nacos服务详情" style="zoom: 67%;" >
        <figcaption>Fig 6. Nacos服务详情.</figcaption>
    </figure>

   查看订阅者列表，可以看到此时还没有订阅者。

    <figure>
        <img src="https://s1.ax1x.com/2023/06/26/pCUsJG8.png" alt="7_Nacos服务订阅者" style="zoom: 67%;" >
        <figcaption>Fig 7. Nacos服务订阅者.</figcaption>
    </figure>   

3. 启动客户端，可以看到返回的response。可以看到所有服务都只选择了同一个服务器，这是由于本例中使用的是Nacos的单机模式。
    <figure>
        <img src="https://s1.ax1x.com/2023/07/02/pCDMRcq.png" alt="7_客户端启动" style="zoom: 67%;" >
        <figcaption>Fig 8. 客户端启动.</figcaption>
    </figure>  

4. 通过查看Nacos控制台，可以看到此时的订阅者列表不再是空。可以查看订阅者的IP地址、端口号以及客户端版本等信息。也可以通过服务列表中的订阅者选项，可以看到具体服务的订阅者信息。
    <figure>
        <img src="https://s1.ax1x.com/2023/06/26/pCUsGPf.png" alt="7_Nacos订阅者列表" style="zoom: 67%;" >
        <figcaption>Fig 9. Nacos订阅者列表.</figcaption>
    </figure>  


# 不足之处

目前的RPC框架还只能在Java语言之间进行数据传输，可以实现跨语言的RPC通信（protobuf）。



