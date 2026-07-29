---
date: 2026-07-16
is_published: true
title: Redis 五种常用数据结构
tags: [Redis, 数据结构, Java]
categories: [Redis]
---

# Redis 五种常用数据结构

目标：根据业务问题选择合适的数据结构，而不是所有东西都塞进字符串。

## 1. String：最基础的数据

适合文本、数字、JSON 字符串：

```redis
SET product:1001:name "机械键盘"
INCR article:1001:view-count
GET article:1001:view-count
```

Java：

```java
jedis.set("product:1001:name", "机械键盘");
jedis.incr("article:1001:view-count");
```

## 2. Hash：一条对象的多个字段

适合用户、商品等字段较少的对象：

```redis
HSET user:42 name "张三" city "上海" level "VIP"
HGET user:42 name
HGETALL user:42
```

Hash 不需要把整个对象序列化成一个 JSON 后再局部修改，因此更新单个字段更方便。

## 3. List：有顺序、允许重复

适合简单队列、最近浏览记录：

```redis
LPUSH user:42:recent-products 1001 1002 1003
LRANGE user:42:recent-products 0 9
```

List 可以做队列，但涉及消息可靠性、消费确认和积压监控时，应优先使用 Kafka、Redis Streams 或专门的消息队列。

## 4. Set：不重复的集合

适合标签、点赞用户、共同好友：

```redis
SADD article:1001:liked-users 42 43 42
SISMEMBER article:1001:liked-users 42
SCARD article:1001:liked-users
```

重复添加 `42` 不会产生重复元素。

## 5. ZSet：带分数的有序集合

适合排行榜、按时间排序的任务：

```redis
ZADD game:rank 98 alice 100 bob 75 carol
ZREVRANGE game:rank 0 2 WITHSCORES
```

分数越大排名越靠前。排行榜更新分数只需再次 `ZADD` 同一个成员。

## 6. 如何选择

| 需求 | 推荐结构 |
|---|---|
| 一个值、计数器、JSON | String |
| 对象字段 | Hash |
| 队列、最近记录 | List |
| 去重、关系集合 | Set |
| 排行、按分数或时间排序 | ZSet |

不要为了“看起来高级”而过度设计。商品详情整体缓存成一个 String JSON，通常就比拆成很多 Hash 和 Set 更容易维护。
