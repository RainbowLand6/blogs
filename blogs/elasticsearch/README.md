---
date: 2026-07-16
is_published: true
title: Elasticsearch Java 小白教程
tags:
  - Elasticsearch
  - Java
  - 搜索
  - 教程
categories:
  - Elasticsearch
---

# Elasticsearch Java 小白教程

这是一套面向 Java 初学者的 Elasticsearch 教程。你会从本地启动一个单节点服务开始，逐步学会索引设计、查询 DSL、Java 客户端、Spring Boot 集成和排障。

本系列约定：

- JDK 17
- Maven 3.9+
- Docker Desktop，建议分配至少 4GB 内存
- Elasticsearch 9.3.0
- 官方 Java API Client 9.3.0

> 第 2 篇为降低本地学习门槛而关闭认证和 HTTPS。该配置只能用于本机开发，绝不能直接用于生产环境。

## 阅读顺序

### 入门

1. [[01-什么是Elasticsearch]]
2. [[02-用Docker启动Elasticsearch]]
3. [[03-索引文档与Mapping]]
4. [[04-查询DSL入门]]

### Java 实战

5. [[05-Java连接Elasticsearch]]
6. [[06-Java实现文档增删改查]]
7. [[07-分页排序与高亮]]
8. [[08-聚合查询与索引设计]]

### 项目集成与排障

9. [[09-Spring-Boot集成Elasticsearch]]
10. [[10-Elasticsearch常见问题与排障清单]]
11. [[11-阿里云Elasticsearch项目实战]]

## 学习项目约定

从第 5 篇开始，Java 示例放在同一个 Maven 项目：

```text
elasticsearch-java-demo/
├── pom.xml
└── src/main/java/com/example/es/
    ├── EsClientFactory.java
    ├── Product.java
    └── ProductRepositoryDemo.java
```

> 学习建议：先用 `curl` 看清请求和返回的 JSON，再用 Java 客户端封装。这样出问题时，能判断是 Elasticsearch 查询问题，还是 Java 代码问题。
