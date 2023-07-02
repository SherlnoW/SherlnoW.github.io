---
layout: post
title: MyRPC-V5-引入注册中心zookeeper
date: 2022-11-28
description: rpc-version5
tag:
- RPC
- Java
---

# 待解决问题

客户端在向服务端发送服务请求时，必须知道服务端的IP地址和端口号，如果服务端自己配置对应服务的IP和端口号，会出现以下问题：

* 客户端没引用一个服务接口，都需要配置对应的服务地址，繁琐且容易出错
* 当服务端出现更换地址、挂掉、新增等情况时，都需要及时通知客户端修改服务端地址，否则会造成服务异常
* 当客户端引用许多服务时，管理杂乱的配置是一个问题

# 解决方案

使用注册中心统一管理配置，当服务端启动后，将IP地址和服务接口存储到注册中心中；客户端在请求服务之前，从注册中心获取对应的服务地址。

常用的注册中心有很多，例如Zookeeper、Eureka、Nacos、Consul等，这里选用Zookeeper作为注册中心。

# 背景知识

**注册中心**

注册中心、客户端、服务端之间的关系如下图所示。
<figure>
    <img src="https://s1.ax1x.com/2023/06/26/pCUsKrd.png" alt="5_注册中心流程" style="zoom: 67%;" >
    <figcaption>Fig 1. 注册中心流程.</figcaption>
</figure>

* 注册中心：负责保存服务端的注册信息，当服务端的服务发生变化时同步变更，客户端感知后刷新本地内存中缓存的服务列表。
* 服务端：启动后，将自身服务注册到注册中心中，并定期发送心跳给注册中心汇报存活状态。
* 客户端：启动后，向注册中心订阅服务，并将返回的服务列表缓存在本地内存中，当注册中心变更后，客户端感知后会刷新本地内存中缓存的服务列表。客户端会基于负载均衡算法，从本地缓存的服务列表中选择一台服务端发起服务调用。

RPC服务注册和调用的具体过程如下：

* 服务端在进行服务注册时，会将服务的IP地址等注册到注册中心，客户端通过该节点的信息定位服务端的具体位置并调用。
* 客户端在第一次调用服务时，通过注册中心找到对应的服务IP地址列表，并缓存到本地供后续使用。当再次调用服务时，直接通过负载均衡算法从IP列表中选取一个服务端调用服务。
* 当服务端的某台服务器宕机或下线时，对应的IP地址会从列表中删除，注册中心也会将新的服务IP地址列表通知给客户端。当某个服务的所有服务器都宕机或下线时，表明该服务已经下线。服务端的某台服务器上线时，注册中心同样会更新IP地址列表并通知客户端。

**常用注册中心介绍与对比**

**Zookeeper**

Zookeeper通过充当一个服务注册表，让多个服务端形成一个集群，让服务端通过服务注册表获取对应的服务地址，即IP地址和端口号。 服务端在Zookeeper中进行服务注册，相当于创建一个节点z，该节点存储着对应服务的IP地址、端口号、传输协议、序列化方式等，客户端通过该节点获取对应的服务端并调用。Zookeeper提供"心跳检测"功能，定时向各个服务端发送一个请求（一个Socket长连接），若超过规定时限没有得到响应，则认为该服务端已经宕机，将其从IP地址列表中删除。客户端通过监听对应路径上的数据变化，判断是否IP地址列表是否发生变化，Zookeeper只会发送一个事件类型和节点信息给对应的客户端，而具体的变更信息需要客户端收到变更通知后自己去获取。

> Zookeeper的缺陷在于客户端发现服务的过程中，因为我们可以容忍注册中心返回的注册信息不是最新的，但不能接受服务端直接宕机造成客户端得不到任何返回的情况发生。即可用性方面的缺陷，这主要是Zookeeper的选举leader选举时间过长导致的。

**Eureka**

Eureka遵循AP原则，具有去中心化架构、请求自动切换、节点间操作复制、自动注册与心跳契约、自动下线、保护模式等特点。Eureka具体的工作流程如下：
* Eureka Server作为注册中心，在启动成功后，等待Eureka Client注册（服务端也是一个Eureka Client），若配置了集群，集群之间会定时同步注册表，实现每个Eureka Server都存在独立完整的服务注册表；
* Eureka Client启动时，根据配置的Eureka Server地址注册服务，之后每30s向Eureka Server发送一次心跳，证明自己服务正常。若90s内Eureka Server没能收到心跳，则会注销该服务实例；
* 若Eureka Server统计到大量的Eureka Client没有送上心跳，则会进入自我保护机制，不再剔除没有送上心跳的Eureka Client。当心跳请求恢复正常后，Eureka Server会自动退出自我保护机制；
* Eureka Client会定时获取服务注册表，并缓存到本地，在服务调用时会先从本地缓存中寻找，若没有，则先从注册中心刷新注册表，再同步到本地缓存；
* Eureka Client获取到目标服务器信息，发起服务调用；
* Eureka Client关闭时向Eureka Server发送取消请求，Eureka Server将其从注册表中删除。

**Nacos**

Nacos主要具有服务发现和服务健康检测、动态服务配置、动态DNS服务等特点，Nacos的注册中心支持CP，也支持AP，可以进行切换。Nacos的实现原理如下：
* 服务端启动后，向Nacos Server的Open Api发起服务注册，将自己的IP地址、端口以及服务名称注册上去，同时服务端与Nacos Server之间建立起心跳机制，用于检测服务状态；
* 客户端查询Nacos Server提供的服务实例列表，并默认每10s就定时拉取依次数据；
* Nacos提供对服务的实时健康检测，会阻止向不健康的服务实例发送请求。当Nacos Server检测到客户端异常，将基于UDP协议推送服务实例列表的更新；
* 客户端可以根据服务实例列表调用对应的服务。

**Consul**

Consul是一款使用Go语言编写的开源工具，内置了服务注册和发现框架、分布一致性协议实现、健康检查、Key/Value存储、多数据中心方案等，具有天然可移植性。具体调用过程如下：
* 当服务端启动后，向Consul发送一个post请求，告知自己的IP地址和端口号；
* 当客户端向服务端发送get请求获取/api/address时，会先从Consul中拿到一个存储IP地址和端口号的tmp表，在获取到对应的IP地址和端口号后在发送get请求。
* tmp表每间隔10s更新，只包含通过了健康检查的服务端。

# 具体实现

### 安装zookeeper

首先安装zookeeper，之后分别打开 /bin目录下的 zkServer.cmd 和 zkCli.cmd，如下图所示。
<figure>
    <img src="https://s1.ax1x.com/2023/06/26/pCUse2D.png" alt="5_zkServer" style="zoom: 67%;" >
    <figcaption>Fig 2. zkServer.</figcaption>
</figure>

<figure>
    <img src="https://s1.ax1x.com/2023/06/26/pCUsmxe.png" alt="5_zkClient" style="zoom: 67%;" >
    <figcaption>Fig 3. zkClient.</figcaption>
</figure>

在 zkCli.cmd中通过`ls /`命令查看目录。

### register包

* **ServiceRegister接口:** 作为服务注册接口，主要负责两个基本功能：注册和查询。
  * `register(String, InetSocketAddress);`：负责注册功能，用于保存服务和地址。
  * `serviceDiscovery(String)`：负责查询功能，用于根据服务名查找地址。

* **ZkServiceRegister类:** 服务注册接口的实现类，通过Curator这个Zookeeper开源的客户端框架实现服务注册和服务发现。

### client包

* **NettyRPCClient类:** 添加ServiceRegister接口字段，修改构造函数，不再手动传入host和port，而是通过ZkServiceRegister类中指定的IP地址和端口号与zookeeper服务端建立连接。
* **TestClient类:** 同样地，修改客户端类，不再传入host和port。
* RPCClient接口、RPCClientProxy类、NettyClientInitializer类、NettyClientHandler类与前一个版本相同。

###  Service包

* **ServiceProvider类:** 需要将服务端的IP地址和端口号传给注册中心，同时在注册中心注册服务。因此修改构造函数，添加host和port两个参数，引入ZkServiceRegister类，并通过register()方法将服务名称和服务端地址在注册中心进行注册。
* UserService接口、UserServiceImpl类、BlogService接口、BolgServiceImpl类与前一个版本相同。

### Server包

* **TestServer类:** 在使用ServiceProvider类时，传入服务端的IP地址和端口号。在暴露服务时已经顺便完成了在注册中心的注册。
* RPCServer接口、NettyServerInitializer类、NettyRPCServerHandler类、NettyRPCServer类与上一个版本相同。

### common包

与上一个版本相同。

### codec包

与上一个版本相同。

# 运行结果

1. 在启动 zkServer.cmd 和 zkCli.cmd后，启动服务端，与zookeeper连接成功，注册中心进行对应处理。
    <figure>
        <img src="https://s1.ax1x.com/2023/06/26/pCUs1at.png" alt="5_服务端启动" style="zoom: 67%;" >
        <figcaption>Fig 3. 服务端启动.</figcaption>
    </figure>

2. 客户端与zookeeper连接成功后，注册中心进行对应处理。
    <figure>
        <img src="https://s1.ax1x.com/2023/06/26/pCUs3IP.png" alt="5_客户端启动" style="zoom: 67%;" >
        <figcaption>Fig 3. 客户端启动.</figcaption>
    </figure>

3. 通过`ls /`指令查看，可以看到此时多了一个节点MyRPC。再通过`ls /MyRPC`指令查看，可以看到所注册服务。
    <figure>
        <img src="https://s1.ax1x.com/2023/06/26/pCUsuKH.png" alt="5_zk" style="zoom: 67%;" >
        <figcaption>Fig 3. zk.</figcaption>
    </figure>

# 不足之处

在客户端根据服务名查询地址时，总是返回第一个IP地址，这对于该IP地址的压力非常大，需要采取负载均衡算法进行分担。