---
date: 2026-07-16
is_published: true
title: Redis 分布式锁入门
tags: [Redis, 分布式锁, Java]
categories: [Redis]
---

# Redis 分布式锁入门

目标：了解 Redis 锁的基本实现和边界，不把它当作解决并发问题的万能钥匙。

## 1. 什么情况需要锁

两个应用实例同时处理“同一订单只能发一次优惠券”时，单机 `synchronized` 无效，因为锁不在同一个 JVM 中。此时才考虑分布式锁。

很多重复问题其实应通过数据库唯一索引或幂等表解决。涉及写库的业务，数据库唯一约束往往比 Redis 锁更可靠、更简单。

## 2. 获取锁：原子地设置值和过期时间

Redis 命令：

```redis
SET lock:coupon:order-1001 request-uuid NX PX 30000
```

| 参数 | 含义 |
|---|---|
| `NX` | Key 不存在时才写入，即只允许一个人拿到锁 |
| `PX 30000` | 锁 30 秒自动过期，防止持有者崩溃后永不释放 |
| `request-uuid` | 当前请求的唯一标识，用于安全释放锁 |

不能把 `SETNX` 和 `EXPIRE` 分成两条命令。它们之间如果程序崩溃，会留下永不过期的锁。

## 3. Java 示例

```java
String lockKey = "lock:coupon:order-1001";
String requestId = java.util.UUID.randomUUID().toString();

try (var jedis = jedisPool.getResource()) {
    String result = jedis.set(lockKey, requestId,
            redis.clients.jedis.params.SetParams.setParams().nx().px(30_000));

    if (!"OK".equals(result)) {
        throw new IllegalStateException("其他请求正在处理");
    }

    // 执行业务逻辑。必须确保正常情况下在锁过期前完成。
}
```

## 4. 释放锁不能直接 `DEL`

以下做法有竞争问题：

```redis
DEL lock:coupon:order-1001
```

如果你的锁已经过期，其他请求拿到了同名新锁，你再执行 `DEL` 会删除别人的锁。

需要“比较 value 后再删除”，并使用 Lua 脚本保证原子性：

```lua
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
end
return 0
```

## 5. 生产建议

手写锁适合学习和极简单的短任务。生产项目优先使用经过验证的客户端封装，并配合：

- 合理的超时时间与超时后的业务补偿。
- 数据库唯一约束或幂等记录。
- 监控锁等待、获取失败和持有时间。

锁只能减少并发进入临界区，不能自动保证支付、库存等业务数据正确。
