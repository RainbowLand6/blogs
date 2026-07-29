---
date: 2026-07-16
is_published: true
title: Dubbo Java 小白教程
tags:
  - Dubbo
  - Java
  - Spring Boot
  - RPC
  - 教程
categories:
  - Dubbo
---

# Dubbo Java 小白教程

这是一套面向 Java 初学者的 Dubbo 教程。主线使用 Spring Boot、Dubbo 3.3.0 和 ZooKeeper 注册中心，完成一个“订单服务调用用户服务”的 RPC 项目。

本系列约定：

- JDK 17
- Maven 3.9+
- Spring Boot 3.x
- Dubbo 3.3.0
- ZooKeeper 3.9+
- Docker Desktop

## 阅读顺序

### 入门

1. [[01-什么是Dubbo]]
2. [[02-用Docker启动ZooKeeper注册中心]]
3. [[03-创建Dubbo多模块项目]]
4. [[04-定义公共接口与DTO]]

### 跑通 RPC 调用

5. [[05-编写Dubbo服务提供者]]
6. [[06-编写Dubbo服务消费者]]
7. [[07-注册中心与负载均衡]]
8. [[08-超时重试异常与幂等]]

### 部署与排障

9. [[09-Dubbo项目部署与配置管理]]
10. [[10-Dubbo常见问题与排障清单]]

## 项目结构

教程中的示例使用三个 Maven 模块：

```text
dubbo-demo/
├── dubbo-api/          # 公共接口、DTO、异常
├── user-provider/      # 用户服务提供者
└── order-consumer/     # 订单服务消费者
```

`dubbo-api` 只放接口契约，不放数据库、Web 或具体业务实现。提供者和消费者共同依赖它，才能保证 RPC 的方法签名一致。

> 学习建议：先让提供者和消费者都在本机跑通，再去理解负载均衡、重试和部署。RPC 的复杂之处不在注解，而在网络、超时、异常和服务治理。
