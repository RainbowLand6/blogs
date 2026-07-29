---
date: 2026-07-16
is_published: true
title: 查询 DSL 入门
tags: [Elasticsearch, Query DSL, 搜索]
categories: [Elasticsearch]
---

# 查询 DSL 入门

目标：掌握全文搜索、精确筛选、范围查询和组合查询。

## 1. 先区分查询和过滤

- **查询（query）**：关心相关度评分，例如搜索“蓝牙耳机”。
- **过滤（filter）**：只关心是否符合条件，例如价格小于 500、品牌是 `SoundGo`。

过滤通常不需要评分，适合放在 `bool.filter` 中。

## 2. 全文搜索：`match`

```json
GET /products/_search
{
  "query": {
    "match": {
      "name": "蓝牙耳机"
    }
  }
}
```

`match` 用于 `text` 字段，会经过字段的分析器处理。

## 3. 精确筛选：`term`

```json
GET /products/_search
{
  "query": {
    "term": {
      "brand": "SoundGo"
    }
  }
}
```

`term` 用于 `keyword`、数字、布尔等精确字段。不要用 `term` 去搜索普通 `text` 商品名。

## 4. 数值范围：`range`

```json
GET /products/_search
{
  "query": {
    "range": {
      "price": {
        "gte": 100,
        "lte": 500
      }
    }
  }
}
```

`gte` 是大于等于，`lte` 是小于等于。

## 5. 组合条件：`bool`

```json
GET /products/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "name": "蓝牙"
          }
        }
      ],
      "filter": [
        {
          "term": {
            "brand": "SoundGo"
          }
        },
        {
          "range": {
            "price": {
              "lte": 500
            }
          }
        }
      ]
    }
  }
}
```

记忆方法：

| 子句 | 含义 |
|---|---|
| `must` | 必须匹配，参与评分 |
| `filter` | 必须符合，不参与评分 |
| `should` | 最好匹配，可提高评分 |
| `must_not` | 必须不符合 |

下一篇把这些 REST 请求改写成 Java API Client 代码。
