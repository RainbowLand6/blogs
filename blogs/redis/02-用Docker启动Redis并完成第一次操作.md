---
date: 2026-07-16
is_published: true
title: 用 Docker 启动 Redis 并完成第一次操作
tags: [Redis, Docker, 入门]
categories: [Redis]
---

# 用 Docker 启动 Redis 并完成第一次操作

目标：本篇结束时，本机有一个运行中的 Redis，并能用命令读写数据。

## 1. 创建 `compose.yaml`

新建一个空目录，在其中创建：

```yaml
services:
  redis:
    image: redis:7-alpine
    container_name: redis-local
    ports:
      - "6379:6379"
    command: redis-server --appendonly yes
```

`--appendonly yes` 会开启 AOF 持久化，适合本地练习。它不等同于生产备份方案，但能避免容器意外重启后所有练习数据立即消失。

## 2. 启动

```bash
docker compose up -d
docker compose ps
```

看到 `redis-local` 状态为 `running` 后，连接 Redis 命令行：

```bash
docker exec -it redis-local redis-cli
```

## 3. 完成第一组命令

在 `redis-cli` 中依次输入：

```redis
PING
SET hello "world"
GET hello
DEL hello
GET hello
```

预期结果：

```text
PONG
OK
"world"
(integer) 1
(nil)
```

## 4. 设置过期时间

```redis
SET verify-code "123456" EX 60
TTL verify-code
GET verify-code
```

`EX 60` 表示这个 Key 60 秒后自动删除。验证码、会话和缓存都常用过期时间。

## 5. 查看 Redis 中有多少 Key

```redis
DBSIZE
```

在本地测试环境可以使用：

```redis
KEYS *
```

但生产环境不要使用 `KEYS *`。它会遍历整个键空间，Key 很多时会阻塞 Redis。生产环境应使用 `SCAN` 分批扫描。

## 6. 停止环境

```bash
docker compose stop
```

需要完全删除容器时再执行：

```bash
docker compose down
```

下一篇用 Java 代码连接本机的 Redis。
