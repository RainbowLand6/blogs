---
date: 2026-07-16
is_published: true
title: Java 实现文档增删改查
tags: [Elasticsearch, Java, CRUD]
categories: [Elasticsearch]
---

# Java 实现文档增删改查

目标：使用 Java API Client 写入、读取、搜索和删除商品文档。

## 1. 定义商品对象

```java
package com.example.es;

public class Product {
    public Long id;
    public String name;
    public String brand;
    public Double price;

    public Product() {
    }

    public Product(Long id, String name, String brand, Double price) {
        this.id = id;
        this.name = name;
        this.brand = brand;
        this.price = price;
    }
}
```

字段使用 `public` 只是为了让入门代码更短；真实项目通常使用私有字段、构造器和 getter/setter。

## 2. 写入或覆盖文档

```java
try (var client = EsClientFactory.create()) {
    Product product = new Product(1001L, "无线蓝牙耳机", "SoundGo", 299.0);

    client.index(request -> request
            .index("products")
            .id(product.id.toString())
            .document(product)
            .refresh(refresh -> refresh.trueValue()));
}
```

同一索引、同一 ID 再次写入时，会覆盖旧文档。示例使用 `refresh=true` 让写入后能立刻搜到；批量写入时不要每条都刷新，会降低写入性能。

## 3. 按 ID 读取

```java
var response = client.get(request -> request
        .index("products")
        .id("1001"), Product.class);

if (response.found()) {
    System.out.println(response.source().name);
}
```

## 4. 按关键词搜索

```java
var response = client.search(request -> request
        .index("products")
        .query(query -> query
                .match(match -> match
                        .field("name")
                        .query("蓝牙耳机"))),
        Product.class);

response.hits().hits().forEach(hit -> {
    Product product = hit.source();
    System.out.println(product.id + " - " + product.name);
});
```

## 5. 删除文档

```java
client.delete(request -> request
        .index("products")
        .id("1001")
        .refresh(refresh -> refresh.trueValue()));
```

## 6. 批量写入建议

大量同步数据时，逐条调用 `index` 网络开销很大，应使用 Bulk API 批量提交。第一版系统先保证数据格式与同步逻辑正确，再根据真实写入量加入批量大小、重试和失败记录。

下一篇解决搜索页面必需的分页、排序和高亮。
