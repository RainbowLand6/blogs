---
date: 2026-07-16
is_published: true
title: Java 使用 Jedis 连接 Redis
tags: [Redis, Java, Jedis]
categories: [Redis]
---

# Java 使用 Jedis 连接 Redis

目标：创建一个 Maven 项目，用 Java 写入并读取 Redis 字符串。

## 1. 创建 Maven 项目

```text
redis-java-demo/
├── pom.xml
└── src/main/java/com/example/redis/
```

`pom.xml`：

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>redis-java-demo</artifactId>
    <version>1.0.0</version>

    <properties>
        <maven.compiler.release>17</maven.compiler.release>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <dependencies>
        <dependency>
            <groupId>redis.clients</groupId>
            <artifactId>jedis</artifactId>
            <version>5.2.0</version>
        </dependency>
    </dependencies>
</project>
```

## 2. 编写第一个 Java 程序

新建 `RedisClient.java`：

```java
package com.example.redis;

import redis.clients.jedis.Jedis;

public class RedisClient {
    public static void main(String[] args) {
        try (Jedis jedis = new Jedis("localhost", 6379)) {
            jedis.setex("user:1:name", 60, "张三");

            String name = jedis.get("user:1:name");
            System.out.println("用户名称：" + name);

            System.out.println("剩余秒数：" + jedis.ttl("user:1:name"));
        }
    }
}
```

运行前确认上一章的 Redis 容器仍在运行。控制台会输出用户名和接近 60 的剩余秒数。

## 3. 为什么使用 try-with-resources

`Jedis` 使用网络连接。`try (...)` 会在代码结束时自动关闭连接，避免连接泄漏。

示例为了易懂，直接创建了一个 `Jedis`。真实 Web 服务不要在每个请求中频繁创建连接，应使用连接池：

```java
import redis.clients.jedis.JedisPool;

JedisPool pool = new JedisPool("localhost", 6379);
try (Jedis jedis = pool.getResource()) {
    jedis.set("app:name", "demo");
}
pool.close();
```

连接池应该在应用启动时创建、应用关闭时统一释放，而不是在每个业务方法里新建。

## 4. 连接失败怎么查

| 异常或现象 | 优先检查 |
|---|---|
| `Connection refused` | Docker 容器是否运行，端口是否是 6379 |
| 读取到 `null` | Key 不存在，或已经过期 |
| 认证失败 | Redis 服务是否设置了密码，客户端是否调用 `auth` |

下一篇认识 Redis 最常用的五种数据结构。
