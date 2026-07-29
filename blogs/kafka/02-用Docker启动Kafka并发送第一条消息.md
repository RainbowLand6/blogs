---
date: 2026-07-16
is_published: true
title: 用 Docker 启动 Kafka 并发送第一条消息
tags: [Kafka, Docker, 入门]
categories: [Kafka]
---

# 用 Docker 启动 Kafka 并发送第一条消息

目标：本篇结束时，本机的 Kafka 可以启动，并能在终端里发送和接收一条消息。

## 1. 准备 Docker

安装并启动 Docker Desktop 后，在终端执行：

```bash
docker version
docker compose version
```

两条命令都能显示版本信息，说明环境可用。

## 2. 新建 `compose.yaml`

在一个空目录创建 `compose.yaml`：

```yaml
services:
  kafka:
    image: apache/kafka:3.9.1
    container_name: kafka-local
    ports:
      - "9092:9092"
```

官方镜像会用默认配置启动单节点 Kafka，足够完成本系列的本地练习。生产环境还需要多个 Broker、高可用、认证和监控。

## 3. 启动 Kafka

```bash
docker compose up -d
docker compose ps
```

看到 `kafka-local` 状态为 `running` 后，再查看最近日志：

```bash
docker compose logs --tail=30 kafka
```

## 4. 创建 Topic

进入容器：

```bash
docker exec -it kafka-local bash
```

创建名为 `hello-kafka` 的 Topic：

```bash
/opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create \
  --topic hello-kafka \
  --partitions 1 \
  --replication-factor 1
```

查看 Topic：

```bash
/opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --list
```

## 5. 发送和接收消息

保持第一个终端不关闭，再打开第二个终端，启动消费者：

```bash
docker exec -it kafka-local /opt/kafka/bin/kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic hello-kafka \
  --from-beginning
```

回到第一个终端，启动生产者：

```bash
/opt/kafka/bin/kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic hello-kafka
```

输入：

```text
hello kafka
```

第二个终端应该立刻显示同样的内容。至此，Kafka 已经跑通。

## 6. 停止环境

```bash
docker compose down
```

`down` 会停止并删除容器。学习过程中想保留 Topic 数据，可以改用：

```bash
docker compose stop
```

## 常见问题

| 现象 | 优先检查 |
|---|---|
| `port is already allocated` | 9092 被其他程序占用，关闭占用程序或换端口 |
| Java 程序连接不上 | 确认 Docker 正在运行，且地址是 `localhost:9092` |
| 容器不断退出 | 执行 `docker compose logs kafka` 查看第一段报错 |
| Topic 已存在 | 这是正常提示，可以直接继续使用它 |

下一篇开始创建 Java 项目，用代码代替命令行生产者和消费者。
