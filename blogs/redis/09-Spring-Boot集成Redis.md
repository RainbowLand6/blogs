---
date: 2026-07-16
is_published: true
title: Spring Boot 集成 Redis
tags: [Redis, Java, Spring Boot]
categories: [Redis]
---

# Spring Boot 集成 Redis

目标：使用 Spring Boot 的 `RedisTemplate` 读写 Redis，并通过注解使用缓存。

## 1. 添加依赖

`pom.xml`：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

Spring Boot 默认使用 Lettuce 客户端，适合大多数应用。不要再额外引入 Jedis，除非项目有明确需要。

## 2. 配置连接

`application.yml`：

```yaml
spring:
  data:
    redis:
      host: localhost
      port: 6379
      timeout: 2s
```

生产环境不要把密码写进仓库，应通过环境变量、密钥管理或部署平台注入。

## 3. 使用 `StringRedisTemplate`

```java
package com.example.demo;

import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Service;

import java.time.Duration;

@Service
public class VerifyCodeService {
    private final StringRedisTemplate redisTemplate;

    public VerifyCodeService(StringRedisTemplate redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    public void save(String phone, String code) {
        redisTemplate.opsForValue()
                .set("verify-code:" + phone, code, Duration.ofMinutes(5));
    }

    public boolean verify(String phone, String code) {
        String cached = redisTemplate.opsForValue().get("verify-code:" + phone);
        return code.equals(cached);
    }
}
```

验证码校验成功后，通常还应删除 Key，避免在过期前被重复使用。

## 4. 使用 `@Cacheable`

启动类加上：

```java
@EnableCaching
```

业务方法：

```java
@Cacheable(cacheNames = "product", key = "#productId")
public Product getProduct(long productId) {
    return productRepository.findById(productId).orElse(null);
}

@CacheEvict(cacheNames = "product", key = "#productId")
public void updateProduct(long productId, Product product) {
    productRepository.save(product);
}
```

这能减少样板代码，但仍要明确序列化方式、TTL、空值缓存和更新一致性。不要因为用了注解就忽略缓存设计。

## 5. 一个常见误解

同一个类内部直接调用带 `@Cacheable` 的方法，缓存注解可能不会生效，因为调用没有经过 Spring 代理。把缓存方法放到独立的 Service，或从其他 Bean 调用，通常更清晰。
