---
date: 2026-07-16
is_published: true
title: 什么是 Dubbo
tags: [Dubbo, Java, RPC, 入门]
categories: [Dubbo]
---

# 什么是 Dubbo

Dubbo 是一个 Java RPC 框架。它让一个服务能够像调用本地方法一样，调用另一个服务提供的方法。

## 1. 什么是 RPC

RPC 是 Remote Procedure Call，意思是远程过程调用。

订单服务需要查询用户信息时，传统 HTTP 调用可能是：

```text
订单服务 -> HTTP 请求 -> 用户服务 -> JSON 响应 -> 订单服务
```

使用 Dubbo 后，代码看起来像：

```java
UserProfile profile = userService.getProfile(userId);
```

但这仍然是一次网络调用。Dubbo 在背后完成服务发现、序列化、网络通信和结果返回。

## 2. Dubbo 的核心角色

| 角色 | 作用 | 示例 |
|---|---|---|
| Provider | 提供服务的一方 | 用户服务 |
| Consumer | 调用服务的一方 | 订单服务 |
| Registry | 记录服务地址的中心 | ZooKeeper |
| Interface | 双方约定的方法契约 | `UserService` |

```text
用户服务（Provider） -> 注册到 ZooKeeper
订单服务（Consumer） -> 从 ZooKeeper 发现用户服务
订单服务（Consumer） -> 通过 Dubbo 调用用户服务
```

## 3. 为什么不直接写 HTTP

HTTP 当然可以用于微服务通信。Dubbo 的价值在于为 Java 服务提供一整套 RPC 能力：

- 基于接口的调用模型。
- 服务注册与发现。
- 客户端负载均衡。
- 超时、重试、容错等治理能力。
- 统一的调用监控和配置。

选择不是“Dubbo 一定比 HTTP 好”，而是看团队技术栈、跨语言需求和治理方式。需要大量跨语言公开 API 时，HTTP/REST 或 gRPC 可能更合适；Java 内部服务调用是 Dubbo 的常见场景。

## 4. 一个重要提醒

Dubbo 调用不是本地方法调用。它可能超时、网络失败、对方服务过载，甚至出现重复调用。因此调用方必须设置合理超时，提供方必须保证接口稳定，写操作还要考虑幂等。

下一篇先启动 ZooKeeper 注册中心。
