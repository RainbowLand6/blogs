---
date: 2026-07-16
is_published: true
title: Kafka Java 小白教程
tags:
  - Kafka
  - Java
  - 消息队列
  - 教程
categories:
  - Kafka
---

# Kafka Java 小白教程

这是一套从零开始的 Kafka 教程。你不需要有消息队列经验；会 Java、Maven 和一点命令行就可以开始。

教程中的环境约定：

- JDK 17
- Maven 3.9+
- Docker Desktop
- Kafka 3.9.1
- Windows、macOS、Linux 均可，命令中的 `docker compose` 相同

学完后，你能独立完成一条常见的业务消息链路：Java 服务发送订单消息，多个消费者按组处理消息，出错时不丢消息、不重复扣费，并知道从哪里开始排障。

## 阅读顺序

### 入门

1. [[01-什么是Kafka]]
2. [[02-用Docker启动Kafka并发送第一条消息]]
3. [[03-创建Java-Maven项目]]
4. [[04-Java生产者发送消息]]
5. [[05-Java消费者消费消息与提交位移]]

### 核心机制

6. [[06-Topic分区与消息Key]]
7. [[07-消费者组与并发消费]]
8. [[08-消息可靠性与幂等消费]]

### 实战与排障

9. [[09-Spring-Boot集成Kafka]]
10. [[10-Kafka常见问题与排障清单]]

## 本系列的练习项目

从第 3 篇开始，所有 Java 示例都放在同一个 Maven 项目中：

```text
kafka-java-demo/
├── pom.xml
└── src/
    └── main/
        └── java/
            └── com/example/kafka/
                ├── SimpleProducer.java
                ├── SimpleConsumer.java
                └── ...
```

> 学习建议：每次只复制并运行当前篇的一个类。先看到控制台结果，再读配置项的含义。Kafka 的概念不少，但第一轮不需要背参数。
