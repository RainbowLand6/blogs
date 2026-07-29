---
date: 2026-07-16
is_published: true
title: 用 Docker 启动 ZooKeeper 注册中心
tags: [Dubbo, ZooKeeper, Docker, 注册中心]
categories: [Dubbo]
---

# 用 Docker 启动 ZooKeeper 注册中心

目标：启动一个本地 ZooKeeper，供后面的 Dubbo Provider 和 Consumer 注册、发现彼此。

## 1. 创建 `compose.yaml`

```yaml
services:
  zookeeper:
    image: zookeeper:3.9
    container_name: dubbo-zookeeper
    ports:
      - "2181:2181"
    environment:
      ZOO_4LW_COMMANDS_WHITELIST: ruok,stat,mntr
```

这是单节点学习环境。生产环境至少需要 ZooKeeper 集群、高可用部署、权限控制和监控。

## 2. 启动并检查

```bash
docker compose up -d
docker compose ps
docker compose logs --tail=50 zookeeper
```

若容器状态为 `running`，再执行：

```bash
docker exec -it dubbo-zookeeper zkServer.sh status
```

单节点通常会显示 `Mode: standalone`。

## 3. Dubbo 使用的地址

后续两个 Spring Boot 项目统一使用：

```text
zookeeper://127.0.0.1:2181
```

应用和 ZooKeeper 都运行在同一个 Docker Compose 网络时，地址应改成：

```text
zookeeper://zookeeper:2181
```

容器内部的 `127.0.0.1` 只指向当前容器自己，不能指向另一台容器。

## 4. 停止

```bash
docker compose stop
```

下一篇创建 Dubbo 多模块项目，让提供者和消费者共享接口。
