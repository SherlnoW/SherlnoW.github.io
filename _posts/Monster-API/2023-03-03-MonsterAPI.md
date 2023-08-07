---
layout: post
title: 【项目】MonsterAPI-项目设计
date: 2023-03-03 13:00
description: MonsterAPI开放平台第一期
tag:
- 项目
- API
- Java
---

# MonsterAPI-项目设计

### 项目介绍

一方面前端开发需要对后端接口进行调用, 另一方面也可以使用现成的系统的功能，所以开发一个提供 API 接口调用的平台，
用户可以注册登录，开通接口调用权限，用户可以使用接口，并且每次调用会进行统计。
管理员可以发布接口、下线接口、接入接口，以及可视化接口的调用情况、数据。

> 示例: http://api.btstu.cn/

API 接口平台：

1. 防止攻击（安全性）
2. 不能随便调用（限制、开通）
3. **统计调用次数**
4. 计费
5. 流量保护
6. API 接口

### 技术选型

##### 前端

- Ant Design Pro
- React
- Ant Design Procompoments
- Umi
- Umi Request (Axios 的封装)

##### 后端

- Java Spring Boot
- Spring Boot Starter （SDK 开发）
 
### 项目计划

1. **初始化展示:** 项目设计、技术选型、基础项目搭建;
2. **接口调用:** 开发调用接口的代码, 并使用 **API签名认证** 保证调用的安全性; 
开发客户端SDK; 管理员接口发布与调用; 接口文档展示、接口在线调用。
3. **接口计费与保护:** 统计用户调用接口次数、计费、开通接口
4. **管理、统计分析:** 提供可视化平台，用图表的方展示所有接口的调用情况，便于调整业务

### 需求分析

1. 管理员可以对接口信息进行增删改查
2. 用户可以访问前台，查看接口信息

### 数据库表设计

##### 用户表

```sql
-- 用户表
create table if not exists api_db.`user`
(
    id           bigint auto_increment comment 'id' primary key,
    userName     varchar(256)                           null comment '用户昵称',
    userAccount  varchar(256)                           not null comment '账号',
    userAvatar   varchar(1024)                          null comment '用户头像',
    gender       tinyint                                null comment '性别',
    userRole     varchar(256) default 'user'            not null comment '用户角色：user / admin',
    userPassword varchar(512)                           not null comment '密码',
    `accessKey` varchar(512) not null comment 'accessKey',
    `secretKey` varchar(512) not null comment 'secretKey',
    createTime   datetime     default CURRENT_TIMESTAMP not null comment '创建时间',
    updateTime   datetime     default CURRENT_TIMESTAMP not null on update CURRENT_TIMESTAMP comment '更新时间',
    isDelete     tinyint      default 0                 not null comment '是否删除',
    constraint uni_userAccount
        unique (userAccount)
) comment '用户';
```

##### 接口信息表

```sql
-- 接口信息表
create table if not exists yuapi.`interface_info`
(
    `id` bigint not null auto_increment comment '主键' primary key,
    `name` varchar(256) not null comment '名称',
    `description` varchar(256) null comment '描述',
    `url` varchar(512) not null comment '接口地址',
    `requestParams` text not null comment '请求参数',
    `requestHeader` text null comment '请求头',
    `responseHeader` text null comment '响应头',
    `status` int default 0 not null comment '接口状态（0-关闭，1-开启）',
    `method` varchar(256) not null comment '请求类型',
    `userId` bigint not null comment '创建人',
    `createTime` datetime default CURRENT_TIMESTAMP not null comment '创建时间',
    `updateTime` datetime default CURRENT_TIMESTAMP not null on update CURRENT_TIMESTAMP comment '更新时间',
    `isDelete` tinyint default 0 not null comment '是否删除(0-未删, 1-已删)'
) comment '接口信息';
```

##### 用户调用接口关系表

```sql
-- 用户调用接口关系表
create table if not exists yuapi.`user_interface_info`
(
    `id` bigint not null auto_increment comment '主键' primary key,
    `userId` bigint not null comment '调用用户 id',
    `interfaceInfoId` bigint not null comment '接口 id',
    `totalNum` int default 0 not null comment '总调用次数',
    `leftNum` int default 0 not null comment '剩余调用次数',
    `status` int default 0 not null comment '0-正常，1-禁用',
    `createTime` datetime default CURRENT_TIMESTAMP not null comment '创建时间',
    `updateTime` datetime default CURRENT_TIMESTAMP not null on update CURRENT_TIMESTAMP comment '更新时间',
    `isDelete` tinyint default 0 not null comment '是否删除(0-未删, 1-已删)'
) comment '用户调用接口关系';
```

## 项目框架

前端：Ant Design Pro 脚手架

后端：Spring Boot 通用模板

## 基础功能

增删改查、登录

前端接口调用：oneapi 插件自动生成





