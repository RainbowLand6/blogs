---
date: 2026-07-16
is_published: true
title: Java 实现缓存旁路模式
tags: [Redis, Java, Cache Aside]
categories: [Redis]
---

# Java 实现缓存旁路模式

目标：实现最常见的 Cache Aside（缓存旁路）模式。

它的读取流程是：

```text
读 Redis -> 命中则返回
        -> 未命中则查数据库 -> 写 Redis（带 TTL）-> 返回
```

## 1. 示例代码

下面用 `Map` 模拟数据库，重点是流程：

```java
package com.example.redis;

import redis.clients.jedis.JedisPool;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

public class ProductCacheService {
    private static final int CACHE_SECONDS = 600;
    private final JedisPool jedisPool;
    private final Map<Long, String> database = new ConcurrentHashMap<>();

    public ProductCacheService(JedisPool jedisPool) {
        this.jedisPool = jedisPool;
        database.put(1001L, "{\"id\":1001,\"name\":\"机械键盘\"}");
    }

    public String getProduct(long productId) {
        String key = "product:" + productId;

        try (var jedis = jedisPool.getResource()) {
            String cached = jedis.get(key);
            if (cached != null) {
                return cached;
            }

            String product = database.get(productId);
            if (product == null) {
                return null;
            }

            jedis.setex(key, CACHE_SECONDS, product);
            return product;
        }
    }

    public void updateProduct(long productId, String productJson) {
        database.put(productId, productJson);

        try (var jedis = jedisPool.getResource()) {
            jedis.del("product:" + productId);
        }
    }
}
```

## 2. 更新数据时为什么删除缓存

更新流程推荐：

```text
更新数据库 -> 删除缓存
```

下一次读取会查到数据库新值，再回填 Redis。

不要使用“先删除缓存，再更新数据库”。两者之间如果有请求进入，可能把旧数据库数据重新写回缓存。

也不建议每次更新都直接写缓存：多个写请求并发时，旧写入可能在后面覆盖新缓存。对大多数业务，更新数据库后删除缓存更简单可靠。

## 3. 空值怎么办

如果不存在的商品 ID 被反复查询，每次都会绕过 Redis 访问数据库。一个基础做法是缓存空值：

```java
jedis.setex("product:" + productId, 60, "");
```

读到空字符串时，直接返回不存在。空值 TTL 应该比正常数据更短，例如 60 秒。

第 7 篇会把这类问题系统化，讲清楚缓存穿透、击穿和雪崩。
