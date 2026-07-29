---
date: 2026-07-16
is_published: true
title: Spring Boot 集成 Elasticsearch
tags: [Elasticsearch, Java, Spring Boot]
categories: [Elasticsearch]
---

# Spring Boot 集成 Elasticsearch

目标：让 Spring Boot 管理 Elasticsearch 客户端，并在业务服务中执行搜索。

## 1. 添加依赖

在 Spring Boot 项目中加入：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
</dependency>
```

使用 Spring Boot 依赖管理时，不要随意再手工指定 Elasticsearch Java Client 版本，避免依赖版本不一致。

## 2. 配置连接

`application.yml`：

```yaml
spring:
  elasticsearch:
    uris: http://localhost:9200
```

这是第 2 篇本地学习环境的 HTTP 配置。生产环境使用 HTTPS，并通过安全方式提供账号、密码或 API Key。

## 3. 注入客户端

```java
package com.example.demo;

import co.elastic.clients.elasticsearch.ElasticsearchClient;
import org.springframework.stereotype.Service;

@Service
public class ProductSearchService {
    private final ElasticsearchClient client;

    public ProductSearchService(ElasticsearchClient client) {
        this.client = client;
    }

    public void search(String keyword) throws Exception {
        var response = client.search(request -> request
                        .index("products")
                        .query(query -> query.match(match -> match
                                .field("name")
                                .query(keyword))),
                ProductDocument.class);

        response.hits().hits().forEach(hit -> {
            ProductDocument product = hit.source();
            System.out.println(product.getName());
        });
    }
}
```

## 4. 数据同步比查询更重要

商品在 MySQL 中更新后，Elasticsearch 索引必须同步更新。常见方案：

| 方案 | 特点 |
|---|---|
| 业务代码双写 | 简单，但失败补偿复杂 |
| 消息队列异步同步 | 解耦，适合多数业务 |
| CDC 订阅数据库变更 | 一致性更强，运维复杂度更高 |

新手项目先明确“数据库是事实来源，ES 是搜索副本”。同步失败要能重试和补偿，不能只依赖一次 HTTP 调用成功。

## 5. 不要把 Repository 当作万能抽象

Spring Data Repository 适合简单的按 ID 读写；复杂搜索、聚合和性能调优时，直接使用 `ElasticsearchClient` 往往更清楚。根据查询复杂度选择，不必强行统一。
