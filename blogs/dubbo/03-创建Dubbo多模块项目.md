---
date: 2026-07-16
is_published: true
title: 创建 Dubbo 多模块项目
tags: [Dubbo, Java, Maven]
categories: [Dubbo]
---

# 创建 Dubbo 多模块项目

目标：建立一个父项目和三个模块，分离接口契约、提供者和消费者。

## 1. 目录结构

```text
dubbo-demo/
├── pom.xml
├── dubbo-api/
│   └── pom.xml
├── user-provider/
│   └── pom.xml
└── order-consumer/
    └── pom.xml
```

## 2. 父项目 `pom.xml`

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>dubbo-demo</artifactId>
    <version>1.0.0</version>
    <packaging>pom</packaging>

    <modules>
        <module>dubbo-api</module>
        <module>user-provider</module>
        <module>order-consumer</module>
    </modules>

    <properties>
        <java.version>17</java.version>
        <spring-boot.version>3.3.0</spring-boot.version>
        <dubbo.version>3.3.0</dubbo.version>
    </properties>
</project>
```

这是用于教学的简化父 POM。真实项目通常还会使用 Spring Boot 的 BOM、统一插件版本、代码检查和依赖收敛规则。

## 3. 三个模块的依赖关系

```text
dubbo-api
   ↑       ↑
   │       │
Provider  Consumer
```

Provider 和 Consumer 都依赖 `dubbo-api`，但它们不应该互相依赖。

## 4. 模块边界

| 模块 | 可以放什么 | 不应放什么 |
|---|---|---|
| `dubbo-api` | 接口、DTO、枚举、业务异常 | 数据库实体、Controller、实现类 |
| `user-provider` | 接口实现、数据库访问 | 订单业务 |
| `order-consumer` | 调用接口、订单业务 | 用户服务实现 |

接口模块要尽量稳定。频繁修改方法签名会让所有调用方都需要升级。

下一篇定义第一个公共接口和 DTO。
