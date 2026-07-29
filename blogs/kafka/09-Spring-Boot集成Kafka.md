---
date: 2026-07-16
is_published: true
title: Spring Boot 集成 Kafka
tags: [Kafka, Java, Spring Boot]
categories: [Kafka]
---

# Spring Boot 集成 Kafka

目标：在 Spring Boot 中发送和监听 Kafka 消息。下面示例基于 Spring Boot 3.x 和 Java 17。

## 1. 添加依赖

在 `pom.xml` 中加入：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-kafka</artifactId>
</dependency>
```

如果项目使用 Spring Boot 的依赖管理，不需要单独指定 Kafka 客户端版本，避免版本不兼容。

## 2. 配置连接

`application.yml`：

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    consumer:
      group-id: coupon-service
      auto-offset-reset: earliest
      enable-auto-commit: false
    listener:
      ack-mode: record
    producer:
      acks: all
      properties:
        enable.idempotence: true
```

`ack-mode: record` 表示监听方法成功返回后，再确认当前记录。监听方法抛异常时不要吞掉异常，否则框架会误以为处理成功。

## 3. 发送消息

```java
package com.example.demo;

import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;

@Service
public class OrderEventProducer {
    private final KafkaTemplate<String, String> kafkaTemplate;

    public OrderEventProducer(KafkaTemplate<String, String> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    public void sendPaidEvent(String orderId) {
        kafkaTemplate.send("order-paid", orderId, "订单已支付：" + orderId);
    }
}
```

真实项目建议发送 JSON 对象，并明确约定字段、版本和 `eventId`。字符串示例只是为了聚焦 Kafka 的基本用法。

## 4. 监听消息

```java
package com.example.demo;

import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Component;

@Component
public class CouponListener {
    @KafkaListener(topics = "order-paid", groupId = "coupon-service")
    public void onOrderPaid(String message) {
        System.out.println("开始发优惠券：" + message);

        // 调用业务服务、写数据库等。
        // 这里成功返回后，Spring Kafka 才会确认本条消息。
    }
}
```

`groupId` 可以写在配置文件，也可以写在注解中。一个项目里建议选一种方式并保持统一；有多个不同业务监听同一 Topic 时，按业务分别使用不同组名。

## 5. 生产环境最容易遗漏的两件事

1. `@KafkaListener` 里的业务要做幂等，不能因为重复投递就重复扣款、重复发券。
2. 异常处理要有明确策略。暂时性失败可以重试；永久失败应进入死信 Topic，并记录足够的上下文。

当基础流程跑通后，再根据项目需求补充 JSON 序列化、批量消费、并发数和死信队列；先把“成功处理后提交”和“重复不出错”做对更重要。
