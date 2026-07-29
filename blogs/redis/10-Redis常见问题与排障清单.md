---
date: 2026-07-16
is_published: true
title: Redis 常见问题与排障清单
tags: [Redis, 排障, 运维]
categories: [Redis]
---

# Redis 常见问题与排障清单

目标：Redis 访问慢、连接失败或内存异常时，先用可验证的信息定位问题。

## 1. 本地环境先确认服务是否存活

```bash
docker compose ps
docker exec -it redis-local redis-cli PING
```

返回 `PONG` 才说明 Redis 本身可以正常响应。

## 2. 常用检查命令

```redis
INFO memory
INFO clients
INFO stats
SLOWLOG GET 10
```

| 命令 | 用途 |
|---|---|
| `INFO memory` | 看内存占用、碎片率、淘汰情况 |
| `INFO clients` | 看连接数、阻塞客户端 |
| `INFO stats` | 看命中率、命令处理量 |
| `SLOWLOG GET 10` | 看最近的慢命令 |

## 3. 常见现象速查

| 现象 | 常见原因 | 第一动作 |
|---|---|---|
| `Connection refused` | Redis 未启动、地址或端口错误 | 检查容器、网络和配置 |
| `NOAUTH` | 服务要求认证 | 确认客户端密码配置 |
| 频繁读到 `null` | Key 不存在、过期、Key 拼错 | 用 `TTL key` 和 `GET key` 验证 |
| 内存持续增长 | Key 无 TTL、大 Value、过期策略不合理 | 看 `INFO memory` 和业务 Key 设计 |
| CPU 高、请求慢 | 大 Key、`KEYS *`、慢 Lua、阻塞命令 | 看 Slow Log，定位命令 |
| 缓存命中率低 | TTL 太短、Key 设计错误、缓存未回填 | 看 `keyspace_hits`、`keyspace_misses` |

## 4. 生产环境不要做的事

- 不执行 `KEYS *`。
- 不把超大 JSON、文件或无限增长的集合放进单个 Key。
- 不在主线程中执行耗时 Lua 脚本。
- 不用 Redis 锁替代数据库约束和业务幂等。
- 不在日志中打印 Redis 密码或完整敏感 Value。

## 5. 排障顺序

1. 确认 Redis 服务存活，应用网络可达。
2. 确认应用连接配置、密码和超时。
3. 用具体 Key 验证是否存在、剩余 TTL 是否正确。
4. 查看应用异常日志和 Redis Slow Log。
5. 检查内存、连接数、命中率、淘汰数。
6. 再分析 Key 设计、缓存策略和容量。

## 6. 学习完成检查

能完成下面四件事，就已经掌握了 Redis 入门主线：

- 用 Docker 启动 Redis 并使用 `redis-cli`。
- Java 使用连接池读写带 TTL 的 Key。
- 使用 Cache Aside 实现“未命中查库并回填”。
- 解释缓存穿透、击穿、雪崩的区别与基础解决办法。

后续可以继续学习 Redis Sentinel、Cluster、主从复制、Stream、Lua 脚本和性能压测；它们都建立在本系列的 Key、TTL、数据结构和缓存一致性基础之上。
