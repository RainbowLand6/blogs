---
date: 2026-07-16
is_published: true
title: 编写 Dubbo 服务消费者
tags: [Dubbo, Java, Consumer, Spring Boot]
categories: [Dubbo]
---

# 编写 Dubbo 服务消费者

目标：让 `order-consumer` 从 ZooKeeper 发现 `UserService`，并在订单逻辑中调用它。

## 1. 添加依赖

消费者的依赖与 Provider 类似：依赖 `dubbo-api`、`dubbo-spring-boot-starter` 和 `dubbo-registry-zookeeper`。Consumer 不需要依赖 Provider 的实现模块。

## 2. 引用远程服务

```java
package com.example.order;

import com.example.dubbo.api.UserProfile;
import com.example.dubbo.api.UserService;
import org.apache.dubbo.config.annotation.DubboReference;
import org.springframework.stereotype.Service;

@Service
public class OrderService {
    @DubboReference(timeout = 1000, check = true)
    private UserService userService;

    public String createOrder(Long userId) {
        UserProfile user = userService.getProfile(userId);
        return "已为 " + user.getNickname() + " 创建订单";
    }
}
```

| 配置 | 含义 |
|---|---|
| `timeout = 1000` | 单次 RPC 最长等待 1 秒 |
| `check = true` | 应用启动时检查是否已有 Provider |

学习环境中 `check=true` 有助于尽早发现服务未注册的问题。生产环境是否启用要看发布顺序和容灾设计；不能用关闭检查来掩盖依赖服务长期不可用。

## 3. 配置注册中心

`application.yml`：

```yaml
spring:
  application:
    name: order-consumer

dubbo:
  application:
    name: order-consumer
  registry:
    address: zookeeper://127.0.0.1:2181
```

## 4. 验证调用

启动顺序：

```text
ZooKeeper -> user-provider -> order-consumer
```

调用 `OrderService.createOrder(1L)` 后，应返回：

```text
已为 用户-1 创建订单
```

现在虽然代码像本地方法调用，但它已经跨进程、跨端口执行。下一篇解释消费者如何从多个 Provider 中选择一个。
