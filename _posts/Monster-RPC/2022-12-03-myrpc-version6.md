---
layout: post
title: 【项目】MyRPC-V6-实现多种负载均衡算法
date: 2022-12-03
description: rpc-version6
tag:
- RPC
- Java
---

# 待解决问题

在客户端从注册中心选择一台服务器的IP地址时，如果只是返回第一个IP地址，会导致该服务器的压力很大，其他的服务器却无事可做。

# 解决方案

引入多种负载均衡算法，以实现将服务请求较为合理地分配到各个支持该服务的服务端上。这里引入轮询负载均衡和随机负载均衡两种算法。

# 背景知识

**轮询负载均衡**

当客户端发送请求后，按照顺序遍历服务列表。该算法可以将请求平均分配到各个服务器上。

**随机负载均衡**

当客户端发送请求后，根据随机数选择对应的服务器。但当请求数不断增加，分配的结果也越来越接近轮询负载均衡。

# 具体实现

### loadbalance包

* **LoadBalance接口:** 负责均衡接口，包含一个balance(List<String\>)方法。
* **RandomLoadBalance类:** 随机负载均衡，随机选择addressList中的一个服务器。
* **RoundLoadBalance类:** 轮询负载均衡，依次选择addressList中的服务器。

### register包

* **ZkServiceRegister类:** 添加LoadBalance接口字段，选择实现该接口的负载均衡类。在serviceDiscovery()方法中，引入所选择的负载均衡器，选择出服务器地址。
* ServiceRegister类

### client包、codec包、common包、service包、server包

与上一版本相同。

# 运行结果

1. 在启动 zkServer.cmd 和 zkCli.cmd后，启动服务端，与zookeeper连接成功，注册中心进行对应处理。
    <figure>
        <img src="https://s1.ax1x.com/2023/06/26/pCUsYRS.png" alt="6_服务端启动" style="zoom: 67%;" >
        <figcaption>Fig 1. 服务端启动.</figcaption>
    </figure>

2. 客户端与zookeeper连接成功后，注册中心进行对应处理，可以看到采用随机负载均衡，选择了不同的服务器。
    <figure>
        <img src="https://s1.ax1x.com/2023/06/26/pCUstxg.png" alt="6_客户端启动" style="zoom: 67%;" >
        <figcaption>Fig 2. 客户端启动.</figcaption>
    </figure>

# 不足之处

目前为止，一个功能较为完整的RPC已经完成，但还有其他许多地方可以进行改进。
