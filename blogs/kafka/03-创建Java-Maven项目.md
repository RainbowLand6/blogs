---
date: 2026-07-16
is_published: true
title: 创建 Java Maven 项目
tags: [Kafka, Java, Maven]
categories: [Kafka]
---

# 创建 Java Maven 项目

目标：创建一个可以连接本地 Kafka 的 Java 项目，并确认依赖下载成功。

## 1. 创建目录

```text
kafka-java-demo/
├── pom.xml
└── src/main/java/com/example/kafka/
```

## 2. 编写 `pom.xml`

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>kafka-java-demo</artifactId>
    <version>1.0.0</version>

    <properties>
        <maven.compiler.release>17</maven.compiler.release>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.apache.kafka</groupId>
            <artifactId>kafka-clients</artifactId>
            <version>3.9.1</version>
        </dependency>
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-simple</artifactId>
            <version>2.0.16</version>
        </dependency>
    </dependencies>
</project>
```

`kafka-clients` 是 Kafka 官方 Java 客户端。`slf4j-simple` 只是让本地运行时能看到必要日志，真实项目通常由 Spring Boot 或公司日志框架统一管理。

## 3. 验证 Maven

在项目根目录运行：

```bash
mvn compile
```

第一次执行会下载依赖，时间稍长。最后看到 `BUILD SUCCESS` 就成功了。

## 4. 连接信息集中写成常量

新建 `KafkaConfig.java`：

```java
package com.example.kafka;

public final class KafkaConfig {
    public static final String BOOTSTRAP_SERVERS = "localhost:9092";
    public static final String TOPIC = "hello-kafka";

    private KafkaConfig() {
    }
}
```

`bootstrap.servers` 是客户端连接 Kafka 的入口地址。开发环境中是本机的 `localhost:9092`；公司环境通常会提供多个 Broker 地址，例如 `kafka-1:9092,kafka-2:9092`。

> 不要把密码、密钥等敏感配置硬编码在 Java 类中。后续接入认证时，应通过环境变量、配置中心或密钥管理服务传入。

项目准备完毕。下一篇用 `KafkaProducer` 发送消息。
