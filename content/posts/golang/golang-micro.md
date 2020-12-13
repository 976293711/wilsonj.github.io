---
title: "Golang微服务概览"
date: 2020-12-03T21:45:53+08:00
draft: false
isCJKLanguage: true
toc: false
images:
tags: 
  - golang
---

## 微服务 
    1. SOA(面向服务)：小即是没、单一职责、尽可能早地创建原型、可移植性比效率更重要
    2. 微服务：服务关注单一业务，服务间采用轻量级的通信机制，可以全自动独立部署。
    3. 微服务优点：原子服务，独立进程，隔离部署，去中心化服务治理。
    4. 缺点：基础设施的建设，复杂度高。 
    5. 串行与并行、分布式一致性

### 组件服务化
   - 多个微服务组合完成了一个完整的用户场景 

### 网关
   - 网关是一种充当转换重任的计算机系统或设备。使用在不同的通信协议、数据格式或语言，甚至体系结构完全不同的两种系统之间，网关是一个翻译器

### 去中心化
     1.数据去中心化 
     2.治理去中心化
     3.技术去中心化

## 微服务设计

### API GateWay（网关） 
Backend for Frontend 面向前端的后端 

Envoy（开源） 专注路由、认证、限流、安全


业务实际流量：
移动端->API Gateway->BFF（B站 用node.js来做服务端渲染 SSR ）->Mircoservice 

### 微服务划分

1. 通过业务职能划分
2. 通过限界上下文来划分（DDD）

#### CQRS
将应用程序分为两部分：命令端和查询端。命令端处理程序创建，更新和删除请求，并在数据更改时发出事件。查询端通过针对一个或多个物化视图执行查询来处理查询，这些物化视图通过订阅数据更改时发出的事件流而保持最新 。

**canal中间件**

订阅mysql的binlog