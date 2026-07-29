---
date: 2026-07-16
is_published: true
title: Kafka 常见问题与排障清单
tags: [Kafka, 排障, 运维]
categories: [Kafka]
---

# Kafka 常见问题与排障清单

目标：消息“没了”或“没被消费”时，按顺序缩小问题范围，而不是先改一堆参数。

## 1. 先画出链路

```text
生产者 -> Kafka Broker -> Topic/Partition -> 消费者组 -> 业务处理 -> 提交位移
```

每次排障先回答：问题卡在哪一段？

## 2. Topic 是否存在，消息是否真的写入

```bash
docker exec -it kafka-local /opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --topic order-paid
```

本地还可用控制台消费者确认消息：

```bash
docker exec -it kafka-local /opt/kafka/bin/kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic order-paid \
  --from-beginning \
  --property print.key=true \
  --property key.separator=" : "
```

若这里也看不到消息，优先检查生产者发送结果和 Broker 日志，不要先怀疑消费者。

## 3. 消费者组积压多少消息

```bash
docker exec -it kafka-local /opt/kafka/bin/kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --group inventory-group
```

关注三列：

| 字段 | 含义 |
|---|---|
| `CURRENT-OFFSET` | 当前组已提交的进度 |
| `LOG-END-OFFSET` | 分区最新消息位置 |
| `LAG` | 两者差值，即待处理数量 |

`LAG` 持续增长：消费者处理慢、报错重试，或根本没在运行。

## 4. 常见现象速查

| 现象 | 常见原因 | 第一动作 |
|---|---|---|
| 连接 `localhost:9092` 失败 | Kafka 未启动、端口错误 | `docker compose ps` 和日志 |
| 生产者超时 | Broker 地址不可达、Topic 元数据获取失败 | 查看生产者异常的根因 |
| 消费者收不到旧消息 | 该组已有位移，或设置为 `latest` | 换新组名验证 |
| 两个消费者没有并行 | 分区数少于消费者数 | 检查 Topic 分区数 |
| 同一消息处理两次 | 提交前进程中断 | 给业务增加幂等保障 |
| 消息堆积 | 下游慢、单条耗时高、异常反复发生 | 看 LAG、应用日志和失败消息 |

## 5. 排障顺序

1. 确认 Broker 存活，网络地址可达。
2. 确认 Topic 存在，分区数量符合预期。
3. 确认生产者拿到了发送成功的回调。
4. 用控制台消费者确认消息在 Topic 中。
5. 查看目标消费者组的 `LAG` 和成员状态。
6. 查看消费者应用日志，特别是反序列化、数据库和下游接口异常。
7. 最后才调整并发、批量大小、超时等性能参数。

## 6. 学习完成检查

完成下面四件事，说明你已经掌握了 Kafka 入门主线：

- 用 Docker 启动本地 Kafka。
- Java 生产者发送带 Key 的消息。
- Java 消费者手动提交位移。
- 两个消费者组分别处理同一 Topic，并能解释为什么会重复消费。

后续可以继续学习 Schema 管理、Kafka Connect、Kafka Streams、集群部署、监控告警和安全认证。它们都建立在本系列讲清楚的 Topic、分区、消费者组、位移和幂等性之上。
