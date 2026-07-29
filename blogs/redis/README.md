---
date: 2026-07-16
is_published: true
title: Redis Java 小白教程
tags:
  - Redis
  - Java
  - 缓存
  - 教程
categories:
  - Redis
---

# Redis Java 小白教程

这是一套从零开始的 Redis 教程。你只需要会 Java、Maven 和一点命令行，就能跟着完成全部示例。

本系列约定：

- JDK 17
- Maven 3.9+
- Docker Desktop
- Redis 7
- Java 客户端：Jedis

学完后，你能在 Java 项目中正确使用 Redis 缓存、设置过期时间、避免常见缓存问题，并能把 Redis 接入 Spring Boot。

## 阅读顺序

### 入门

1. [[01-什么是Redis]]
2. [[02-用Docker启动Redis并完成第一次操作]]
3. [[03-Java使用Jedis连接Redis]]
4. [[04-Redis五种常用数据结构]]

### 缓存实战

5. [[05-过期时间与持久化]]
6. [[06-Java实现缓存旁路模式]]
7. [[07-缓存穿透击穿雪崩]]
8. [[08-Redis分布式锁入门]]

### 项目集成与排障

9. [[09-Spring-Boot集成Redis]]
10. [[10-Redis常见问题与排障清单]]

## 学习项目约定

从第 3 篇开始，Java 示例都可以放到同一个 Maven 项目：

```text
redis-java-demo/
├── pom.xml
└── src/main/java/com/example/redis/
    ├── RedisClient.java
    ├── DataTypeDemo.java
    └── ProductCacheService.java
```

> 学习建议：先让每段代码跑出结果，再理解配置。Redis 命令很短，但“数据应存多久、失效后如何回源”比命令本身更重要。
