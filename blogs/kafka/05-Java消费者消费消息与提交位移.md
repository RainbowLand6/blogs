---
date: 2026-07-16
is_published: true
title: Java 消费者消费消息与提交位移
tags: [Kafka, Java, Consumer, Offset]
categories: [Kafka]
---

# Java 消费者消费消息与提交位移

目标：用 Java 接收消息，并理解消费者为什么能在重启后从上次位置继续。

## 1. 编写消费者

新建 `SimpleConsumer.java`：

```java
package com.example.kafka;

import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.apache.kafka.common.serialization.StringDeserializer;

import java.time.Duration;
import java.util.List;
import java.util.Properties;

public class SimpleConsumer {
    public static void main(String[] args) {
        Properties properties = new Properties();
        properties.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, KafkaConfig.BOOTSTRAP_SERVERS);
        properties.put(ConsumerConfig.GROUP_ID_CONFIG, "demo-order-group");
        properties.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        properties.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        properties.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        properties.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);

        try (KafkaConsumer<String, String> consumer = new KafkaConsumer<>(properties)) {
            consumer.subscribe(List.of(KafkaConfig.TOPIC));

            while (true) {
                ConsumerRecords<String, String> records = consumer.poll(Duration.ofSeconds(1));

                records.forEach(record -> {
                    System.out.printf(
                            "收到消息：key=%s, value=%s, partition=%d, offset=%d%n",
                            record.key(), record.value(), record.partition(), record.offset()
                    );
                });

                if (!records.isEmpty()) {
                    consumer.commitSync();
                }
            }
        }
    }
}
```

先运行消费者，再运行上一章的生产者。消费者会打印收到的消息。停止消费者后重新运行，它不会重复打印已经成功提交的消息。

## 2. Offset 是什么

Offset（位移）是消息在一个分区里的连续编号：

```text
Partition 0: [offset 0] [offset 1] [offset 2] ...
```

消费者组会记录自己已处理到哪里。`commitSync()` 的作用是：告诉 Kafka，“这一批消息已经处理完成，可以从后面继续了”。

## 3. 为什么先处理、再提交

推荐顺序是：

```text
拉取消息 -> 处理业务 -> 提交位移
```

如果先提交、再处理，程序在业务处理前崩溃，Kafka 会以为消息已经完成，消息可能丢失。

反过来，先处理后提交时，如果处理成功但提交前崩溃，重启后可能再次收到该消息。因此消费者业务必须具备幂等性，第 8 篇会详细讲。

## 4. `auto.offset.reset` 的作用

这个参数只在“该消费者组从未提交过位移”时才生效：

| 配置 | 含义 |
|---|---|
| `earliest` | 从该 Topic 仍保留的最早消息开始 |
| `latest` | 只接收启动后的新消息 |
| `none` | 没有历史位移时直接报错 |

学习时用 `earliest` 最直观；生产环境要根据业务是否需要补历史数据决定。

## 5. 想重新消费怎么办

最简单的方法是换一个新的 `group.id`。Kafka 会把它当作一个全新消费者组，并按 `earliest` 重新读取保留的消息。

下一篇讲 Topic 为什么要分区，以及 Key 如何影响消息顺序。
