---
date: 2026-07-16
is_published: true
title: Topic 分区与消息 Key
tags: [Kafka, Partition, Key]
categories: [Kafka]
---

# Topic 分区与消息 Key

目标：理解 Kafka 的并发能力来自哪里，以及哪些消息能够保证顺序。

## 1. Topic 为什么有分区

一个 Topic 可以拆成多个 Partition：

```text
Topic: order-paid
├── Partition 0: 0, 1, 2, 3 ...
├── Partition 1: 0, 1, 2, 3 ...
└── Partition 2: 0, 1, 2, 3 ...
```

多个分区能由多个消费者并行处理，提高吞吐量。代价是：Kafka 只保证**同一个分区内**的消息顺序，不保证整个 Topic 的全局顺序。

## 2. 创建三分区 Topic

```bash
docker exec -it kafka-local /opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create \
  --topic order-paid \
  --partitions 3 \
  --replication-factor 1
```

## 3. Key 如何决定分区

没有指定分区时，Kafka 会根据 Key 计算目标分区。相同 Key 会进入同一分区。

```java
new ProducerRecord<>("order-paid", "order-1001", "订单 1001 支付成功");
new ProducerRecord<>("order-paid", "order-1001", "订单 1001 已发货");
```

这两条消息会被发到同一分区，顺序得以保留。

## 4. 应该使用什么 Key

| 场景 | 推荐 Key | 目的 |
|---|---|---|
| 订单状态流转 | 订单号 | 同一订单状态有序 |
| 用户行为 | 用户 ID | 同一用户事件有序 |
| 设备上报 | 设备 ID | 同一设备数据有序 |
| 无关联日志 | 可不传 Key | 平均分配负载 |

## 5. 不要滥用单一 Key

如果所有消息都使用 `order` 作为 Key，所有消息都会进入同一个分区，三个分区的并发能力只用到一个。一个热点 Key 也会导致分区倾斜。

选择 Key 的原则很朴素：

1. 需要相对顺序的一组消息，使用同一个 Key。
2. 不同业务实体尽量分散到不同 Key。
3. 不要依赖跨分区的先后顺序。

## 6. 分区数和消费者数的关系

同一个消费者组中，一个分区同一时刻只能分配给一个消费者：

```text
3 个分区 + 2 个消费者 -> 两个消费者都能分到工作
3 个分区 + 5 个消费者 -> 最多只有 3 个消费者工作
```

这就是后面消费者组并发设计的基础。
