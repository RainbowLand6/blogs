---
date: 2026-07-16
is_published: true
title: Java 连接 Elasticsearch
tags: [Elasticsearch, Java, Client]
categories: [Elasticsearch]
---

# Java 连接 Elasticsearch

目标：创建 Maven 项目并使用官方 Java API Client 连接本地 Elasticsearch。

## 1. `pom.xml`

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>elasticsearch-java-demo</artifactId>
    <version>1.0.0</version>

    <properties>
        <maven.compiler.release>17</maven.compiler.release>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <dependencies>
        <dependency>
            <groupId>co.elastic.clients</groupId>
            <artifactId>elasticsearch-java</artifactId>
            <version>9.3.0</version>
        </dependency>
    </dependencies>
</project>
```

运行：

```bash
mvn compile
```

首次运行会下载依赖；看到 `BUILD SUCCESS` 即表示项目准备完成。

## 2. 创建客户端

新建 `EsClientFactory.java`：

```java
package com.example.es;

import co.elastic.clients.elasticsearch.ElasticsearchClient;

public final class EsClientFactory {
    private EsClientFactory() {
    }

    public static ElasticsearchClient create() {
        return ElasticsearchClient.of(builder -> builder
                .host("http://localhost:9200"));
    }
}
```

这个客户端对应第 2 篇关闭认证的本地环境。生产环境应使用 `https`，并通过 API Key 或账号密码配置认证与 CA 证书校验。

## 3. 发起健康检查

```java
package com.example.es;

import co.elastic.clients.elasticsearch.ElasticsearchClient;

public class PingDemo {
    public static void main(String[] args) throws Exception {
        try (ElasticsearchClient client = EsClientFactory.create()) {
            boolean available = client.ping().value();
            System.out.println("Elasticsearch 可用：" + available);
        }
    }
}
```

看到 `Elasticsearch 可用：true`，说明 Java 程序已经连接成功。

客户端应在应用启动时创建并复用，应用关闭时统一关闭；不要每次请求都新建一个客户端。

下一篇使用它实现商品文档的增删改查。
