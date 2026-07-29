---
date: 2026-07-16
is_published: true
title: 编写 Dubbo 服务提供者
tags: [Dubbo, Java, Provider, Spring Boot]
categories: [Dubbo]
---

# 编写 Dubbo 服务提供者

目标：让 `user-provider` 把 `UserService` 注册到 ZooKeeper。

## 1. 添加依赖

`user-provider/pom.xml` 的核心依赖：

```xml
<dependencies>
    <dependency>
        <groupId>com.example</groupId>
        <artifactId>dubbo-api</artifactId>
        <version>1.0.0</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-spring-boot-starter</artifactId>
        <version>3.3.0</version>
    </dependency>
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-registry-zookeeper</artifactId>
        <version>3.3.0</version>
    </dependency>
</dependencies>
```

项目还需要正常的 Spring Boot 依赖和插件配置。这里仅展示 Dubbo 相关部分。

## 2. 实现并暴露服务

```java
package com.example.user;

import com.example.dubbo.api.UserProfile;
import com.example.dubbo.api.UserService;
import org.apache.dubbo.config.annotation.DubboService;

@DubboService
public class UserServiceImpl implements UserService {
    @Override
    public UserProfile getProfile(Long userId) {
        return new UserProfile(userId, "用户-" + userId, "VIP");
    }
}
```

`@DubboService` 表示这个实现要作为 Dubbo 服务对外暴露，不是 Spring MVC 的 HTTP Controller。

## 3. 配置 `application.yml`

```yaml
spring:
  application:
    name: user-provider

dubbo:
  application:
    name: user-provider
  registry:
    address: zookeeper://127.0.0.1:2181
  protocol:
    name: dubbo
    port: -1
```

`port: -1` 让 Dubbo 自动选择可用端口，适合本地学习。生产环境通常应明确端口、网卡和注册 IP，避免多网卡、容器网络造成错误注册。

## 4. 启动验证

先启动第 2 篇的 ZooKeeper，再启动 `user-provider`。日志中应该能看到服务导出和注册中心连接成功的信息。

不要只看“应用启动成功”。Provider 必须成功注册到 ZooKeeper，Consumer 才能发现它。

下一篇编写订单服务消费者并完成第一次远程调用。
