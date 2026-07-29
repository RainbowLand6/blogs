---
date: 2026-07-16
is_published: true
title: 阿里云 Elasticsearch 项目实战
tags: [Elasticsearch, 阿里云, Java, Spring Boot, 项目实战]
categories: [Elasticsearch]
---

# 阿里云 Elasticsearch 项目实战

目标：把商品搜索从本地 Docker 环境迁移到阿里云 Elasticsearch，并完成“数据库商品数据同步到搜索索引、提供搜索接口、上线检查”这条最小闭环。

本文假设：

- MySQL 是商品数据的事实来源。
- 阿里云 Elasticsearch 是商品搜索副本。
- 应用服务和 Elasticsearch 位于同一个 VPC，优先使用 VPC 地址访问。
- 项目使用 Spring Boot 3.x、Java 17。

> 控制台中实例版本、访问地址和安全项的名称可能随产品版本调整。以实例详情页展示的 VPC 地址、访问端口、账号和安全配置为准。

## 1. 项目架构

```text
运营后台 -> 商品服务 -> MySQL
                    └-> 商品变更事件 -> 同步消费者 -> 阿里云 Elasticsearch

用户搜索 -> 搜索服务 -> 阿里云 Elasticsearch
```

不要让搜索接口直接查询 MySQL 再拼凑搜索结果。搜索与业务写入的职责不同：MySQL 负责正确保存商品，Elasticsearch 负责快速检索和筛选商品。

## 2. 阿里云侧准备

在阿里云 Elasticsearch 实例详情中完成以下准备：

1. 记录实例的 **VPC 访问地址** 和端口。
2. 创建或重置供应用使用的账号密码，并按最小权限授权。
3. 在访问白名单或安全组中放行应用所在的 VPC 网段或安全组。
4. 确认应用服务器可以解析并访问该 VPC 地址。
5. 为生产实例启用 HTTPS，并保存平台提供的 CA 证书或证书链信息。

不要为了“先跑通”把 Elasticsearch 暴露到公网，也不要将 `elastic` 超级用户账号直接写入业务服务配置。

## 3. 使用环境变量保存连接信息

部署环境中设置：

```text
ES_URIS=https://your-es-vpc-endpoint:9200
ES_USERNAME=app_search
ES_PASSWORD=replace-with-secret
```

本地开发可放入未提交的环境变量文件或 IDE 运行配置。生产环境应由部署平台、密钥管理服务或 Kubernetes Secret 注入。

`application.yml`：

```yaml
spring:
  elasticsearch:
    uris: ${ES_URIS}
    username: ${ES_USERNAME}
    password: ${ES_PASSWORD}
    connection-timeout: 2s
    socket-timeout: 5s
```

如果实例使用平台提供的私有 CA，应用 JVM 必须信任该 CA。不要为了绕过证书问题关闭 HTTPS 校验。

## 4. 创建商品索引

先在测试环境执行以下 Mapping，确认字段设计后再推广到生产：

```json
PUT /products_v1
{
  "mappings": {
    "properties": {
      "id": { "type": "long" },
      "name": {
        "type": "text",
        "fields": {
          "keyword": { "type": "keyword" }
        }
      },
      "brand": { "type": "keyword" },
      "categoryId": { "type": "long" },
      "price": { "type": "double" },
      "sales": { "type": "integer" },
      "status": { "type": "keyword" },
      "updatedAt": { "type": "date" }
    }
  }
}
```

使用版本化索引名 `products_v1` 的好处是：需要修改 Mapping 时，可以新建 `products_v2`、全量同步数据、验证通过后再切换别名，避免直接在线修改旧索引。

创建别名：

```json
POST /_aliases
{
  "actions": [
    {
      "add": {
        "index": "products_v1",
        "alias": "products"
      }
    }
  ]
}
```

之后业务查询统一访问 `products`，而不是写死 `products_v1`。

## 5. 商品数据同步

最容易理解的同步流程是“全量初始化 + 增量事件”：

```text
首次上线：MySQL 全量读取 -> Bulk 写入 products_v1
日常更新：商品变更 -> 发布事件 -> 消费事件 -> 更新 products
失败补偿：定时扫描 MySQL -> 修复缺失或过期文档
```

### 5.1 增量同步示例

商品更新成功后，发布一条事件：

```json
{
  "eventId": "74d9c7f6-f4bf-440e-9f2b-5c40da64e6ac",
  "productId": 1001,
  "eventType": "PRODUCT_UPDATED"
}
```

消费者收到事件后，从 MySQL 查询最新商品数据，再写入 Elasticsearch。不要只把事件里的局部字段直接更新到 ES；查询数据库最新状态能降低消息乱序和重复投递带来的风险。

```java
package com.example.search;

import co.elastic.clients.elasticsearch.ElasticsearchClient;
import org.springframework.stereotype.Service;

@Service
public class ProductIndexService {
    private final ElasticsearchClient client;
    private final ProductRepository productRepository;

    public ProductIndexService(
            ElasticsearchClient client,
            ProductRepository productRepository) {
        this.client = client;
        this.productRepository = productRepository;
    }

    public void syncProduct(long productId) throws Exception {
        Product product = productRepository.findById(productId)
                .orElseThrow(() -> new IllegalStateException("商品不存在：" + productId));

        client.index(request -> request
                .index("products")
                .id(Long.toString(product.getId()))
                .document(ProductDocument.from(product)));
    }

    public void deleteProduct(long productId) throws Exception {
        client.delete(request -> request
                .index("products")
                .id(Long.toString(productId)));
    }
}
```

`ProductDocument.from(product)` 的职责是把数据库实体转换成搜索文档。不要把 JPA 实体直接当 ES 文档使用，避免数据库字段变更无意间影响搜索 Mapping。

### 5.2 全量同步要使用 Bulk API

全量同步时不要循环单条 `index` 请求。应按固定批次（例如每批 500 到 2000 条，实际值按文档大小和集群负载压测确定）构造 Bulk 请求，并记录失败的商品 ID 以便重试。

全量同步完成后，至少比较：

- MySQL 中上架商品数。
- Elasticsearch 索引文档数。
- 随机抽样商品的关键字段。
- 核心关键词的搜索结果。

## 6. 提供搜索接口

一个简化的商品搜索接口可接收关键词、品牌、价格区间和页码：

```java
package com.example.search;

import co.elastic.clients.elasticsearch.ElasticsearchClient;
import org.springframework.stereotype.Service;

@Service
public class ProductSearchService {
    private final ElasticsearchClient client;

    public ProductSearchService(ElasticsearchClient client) {
        this.client = client;
    }

    public void search(String keyword, String brand, double minPrice, double maxPrice)
            throws Exception {
        var response = client.search(request -> request
                        .index("products")
                        .query(query -> query.bool(bool -> {
                            if (keyword != null && !keyword.isBlank()) {
                                bool.must(must -> must.match(match -> match
                                        .field("name")
                                        .query(keyword)));
                            }

                            if (brand != null && !brand.isBlank()) {
                                bool.filter(filter -> filter.term(term -> term
                                        .field("brand")
                                        .value(brand)));
                            }

                            bool.filter(filter -> filter.range(range -> range
                                    .number(number -> number
                                            .field("price")
                                            .gte(minPrice)
                                            .lte(maxPrice))));
                            return bool;
                        })),
                ProductDocument.class);

        response.hits().hits().forEach(hit -> {
            ProductDocument product = hit.source();
            System.out.println(product.getName());
        });
    }
}
```

接口层还应做参数校验：

- `pageSize` 设置上限，例如不超过 100。
- 关键词长度设置上限，拒绝异常长的输入。
- 价格范围必须合法。
- 不对用户输入开放任意字段名、脚本和通配符查询。

## 7. 上线检查清单

### 网络与安全

- 应用使用 VPC 地址而不是公网地址。
- 白名单或安全组只放行必要来源。
- 应用使用专用账号，权限遵循最小化原则。
- 密码和证书不进入 Git 仓库、日志或异常响应。

### 数据正确性

- MySQL 到 ES 的同步事件可重试，且同一商品重复同步不会产生重复文档。
- 删除、下架商品能从搜索索引中移除或过滤。
- 有全量重建和差异修复能力。
- Mapping 已在测试环境验证，字段类型不依赖动态猜测。

### 可观测性

- 监控搜索请求耗时、错误率、连接失败次数。
- 监控集群健康、JVM 堆、磁盘和分片状态。
- 记录同步失败的事件 ID、商品 ID 和异常原因。
- 为积压消息、集群 `red` 状态和磁盘空间设置告警。

## 8. 实战小结

阿里云 Elasticsearch 项目的关键不在于“连上一个云地址”，而在于把它当成数据库的搜索副本来管理：

1. 网络、账号、证书配置正确且不泄露。
2. 索引 Mapping 能支撑实际搜索、筛选和聚合需求。
3. MySQL 到 ES 的同步可重试、可补偿、可重建。
4. 搜索接口控制分页与查询复杂度，并具备监控告警。

先做出这条最小闭环，再根据真实业务继续补充中文分词、同义词、排序策略、索引生命周期和高可用方案。
