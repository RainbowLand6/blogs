---
date: 2026-07-16
is_published: true
title: 索引、文档与 Mapping
tags: [Elasticsearch, Index, Mapping]
categories: [Elasticsearch]
---

# 索引、文档与 Mapping

目标：主动创建一个 `products` 索引，明确字段类型，而不是让 Elasticsearch 猜测一切。

## 1. 为什么要先设计 Mapping

Elasticsearch 可以动态推断字段类型，但第一次写入的数据一旦推断错误，之后修改很麻烦。

例如商品名称应该支持全文搜索，品牌应该支持精确过滤和聚合，两者不能只用一种字段类型。

## 2. 创建索引

```json
PUT /products
{
  "mappings": {
    "properties": {
      "id": {
        "type": "long"
      },
      "name": {
        "type": "text",
        "fields": {
          "keyword": {
            "type": "keyword"
          }
        }
      },
      "brand": {
        "type": "keyword"
      },
      "price": {
        "type": "double"
      },
      "stock": {
        "type": "integer"
      },
      "createdAt": {
        "type": "date"
      }
    }
  }
}
```

PowerShell 可以把上面的 JSON 保存为字符串后用 `Invoke-RestMethod -Method Put` 发到 `http://localhost:9200/products`；也可以用 Kibana Dev Tools 执行。学习阶段先理解请求结构即可。

## 3. 最常用的字段类型

| 类型 | 适用场景 | 示例 |
|---|---|---|
| `text` | 分词后的全文搜索 | 商品名、文章正文 |
| `keyword` | 精确匹配、聚合、排序 | 品牌、状态、订单号 |
| `long`、`integer` | 整数 | 库存、数量 |
| `double` | 小数 | 价格、评分 |
| `boolean` | true/false | 是否上架 |
| `date` | 时间 | 创建时间 |

## 4. `text` 和 `keyword` 的区别

`text` 会经过分析器处理，适合 `match` 全文搜索。`keyword` 把整个值视为一个完整词，适合 `term` 精确筛选和聚合。

```text
name: "无线蓝牙耳机"
name        -> 用于搜索“蓝牙”
name.keyword -> 用于按完整商品名排序或精确过滤
```

不要对 `text` 字段直接做聚合；通常使用它的 `.keyword` 子字段。

## 5. 写入文档

```json
PUT /products/_doc/1001
{
  "id": 1001,
  "name": "无线蓝牙耳机",
  "brand": "SoundGo",
  "price": 299.0,
  "stock": 80,
  "createdAt": "2026-07-16T10:00:00Z"
}
```

`1001` 是文档 ID。业务中使用稳定的商品 ID 作为文档 ID，重复同步时会覆盖同一商品，而不是制造重复文档。

下一篇开始查询 DSL，真正搜索这批数据。
