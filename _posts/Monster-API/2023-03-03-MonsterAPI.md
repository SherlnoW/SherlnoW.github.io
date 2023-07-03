---
layout: post
title: MonsterAPI开放平台
date: 2023-03-03 13:00
description: MonsterAPI开放平台
tag:
- 项目
- API
- Java
---

# API 开放平台

## 第一期

项目背景：

1. 前端开发需要用到后端接口
2. 使用现成的系统的功能（http://api.btstu.cn/）

API 接口平台：

1. 防止攻击（安全性）
2. 不能随便调用（限制、开通）
3. **统计调用次数**
4. 计费
5. 流量保护
6. API 接口

## 项目介绍

做一个提供 API 接口调用的平台，用户可以注册登录，开通接口调用权限，用户可以使用接口，并且每次调用会进行统计。管理员可以发布接口、下线接口、接入接口，以及可视化接口的调用情况、数据。

## 技术选型

### 前端

- Ant Design Pro
- React
- Ant Design Procompoments
- Umi
- Umi Request (Axios 的封装)

### 后端

- Java Spring Boot
- Sring Boot Starter （SDK 开发）

## 项目计划

### 第一期 ---- 初始化展示

项目介绍、设计、技术选型

基础项目搭建

接口管理

用户查看接口

### 第二期 ---- 接口调用

1. 继续开发接口管理前端页面
2. 开发调用这个接口的代码
3. 保证调用的安全性**（API 签名认证）**
4. 客户端 SDK  的开发
5. 管理员接口 **发布** 与调用
6. 接口文档展示、接口在线调用

### 第三期 ----- 接口计费与保护

统计用户调用接口次数

限流

计费

日志

开通

### 第四期 ---- 管理、统计分析

提供可视化平台，用图表的方展示所有接口的调用情况，便于调整业务

**实现预警**

## 需求分析

1. 管理员可以对接口信息进行增删改查
2. 用户可以访问前台，查看接口信息

## 数据库表设计

### 接口信息表

```sql
-- 接口信息表
create table if not exists mofeng.`interface_info`
(
    `id` bigint not null auto_increment comment '主键' primary key,
    `name` varchar(256) not null comment '用户名',
    `description` varchar(256) null comment '描述',
    `url` varchar(512) not null comment '接口地址',
    `requestHeader` text null comment '请求头',
    `responseHeader` text null comment '响应头',
    `userId` varchar(256) not null comment '创建人',
    `status` int default 0 not null comment '接口状态（0 - 关闭， 1 - 开启））',
    `method` varchar(256) not null comment '请求类型',
    `createTime` datetime default CURRENT_TIMESTAMP not null comment '创建时间',
    `updateTime` datetime default CURRENT_TIMESTAMP not null on update CURRENT_TIMESTAMP comment '更新时间',
    `isDelete` tinyint default 0 not null comment '是否删除(0-未删, 1-已删)'
    ) comment '接口信息表';
```

## 项目脚手架

前端：Ant Design Pro 脚手架

后端：Spring Boot 通用模板

## 基础功能

增删改查、登录（复制、粘贴）

前端接口调用：oneapi 插件自动生成

openapi  的规范





