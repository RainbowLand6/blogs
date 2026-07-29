---
date: 2026-07-16
is_published: true
title: Java 生产者发送消息
tags: [Kafka, Java, Producer]
categories: [Kafka]
---

# Java 生产者发送消息

目标：用 Java 向 `hello-kafka` Topic 发送一条带 Key 的消息，并确认 Kafka 已经接收。

## 1. 编写生产者

新建 `SimpleProducer.java`：

```java
package com.example.kafka;

import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.ProducerConfig;
import org.apache.kafka.clients.producer.ProducerRecord;
import org.apache.kafka.clients.producer.RecordMetadata;
import org.apache.kafka.common.serialization.StringSerializer;

import java.util.Properties;

public class SimpleProducer {
    public static void main(String[] args) throws Exception {
        Properties properties = new Properties();
        properties.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, KafkaConfig.BOOTSTRAP_SERVERS);
        properties.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
        properties.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
        properties.put(ProducerConfig.ACKS_CONFIG, "all");

        try (KafkaProducer<String, String> producer = new KafkaProducer<>(properties)) {
            ProducerRecord<String, String> record =
                    new ProducerRecord<>(KafkaConfig.TOPIC, "order-1001", "支付成功");

            RecordMetadata metadata = producer.send(record).get();
            System.out.printf(
                    "发送成功：topic=%s, partition=%d, offset=%d%n",
                    metadata.topic(), metadata.partition(), metadata.offset()
            );
        }
    }
}
```

## 2. 运行

先确认第 2 篇的 Kafka 容器正在运行，然后在 IDE 中运行 `SimpleProducer.main`。输出类似：

```text
发送成功：topic=hello-kafka, partition=0, offset=0
```

也可以用 Maven 运行：

```bash
mvn compile exec:java -Dexec.mainClass=com.example.kafka.SimpleProducer
```

如果未配置 Maven Exec 插件，直接从 IDE 运行更简单。

## 3. 代码逐行理解

| 代码 | 含义 |
|---|---|
| `StringSerializer` | 把 Java 的 `String` 转成 Kafka 能传输的字节数组 |
| `ProducerRecord` | 一条待发送的消息，包含 Topic、Key、Value |
| `producer.send(...)` | 把消息交给客户端发送，返回异步结果 |
| `.get()` | 等待 Broker 确认结果，方便学习与排错 |
| `RecordMetadata` | Broker 返回的实际分区和位移 |

## 4. 为什么要写 Key

示例里的 `order-1001` 是消息 Key。相同 Key 的消息会被发送到同一个分区，因此可以保证这个订单相关消息的先后顺序。

不要用随机 UUID 当业务顺序的 Key；如果希望同一订单有序，就用订单号。

## 5. 一个容易踩的坑

下面代码看起来能运行，但有风险：

```java
producer.send(record);
```

程序可能在消息真正发出前结束。实际项目至少应在关闭前调用 `flush()`，更常见的做法是像示例一样通过 try-with-resources 自动关闭生产者；关闭时会完成已提交的发送请求。

下一篇会消费这条消息，并解释“消费到哪里了”是如何记录的。
