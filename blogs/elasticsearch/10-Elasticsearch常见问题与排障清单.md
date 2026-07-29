---
date: 2026-07-16
is_published: true
title: Elasticsearch 常见问题与排障清单
tags: [Elasticsearch, 排障, 运维]
categories: [Elasticsearch]
---

# Elasticsearch 常见问题与排障清单

目标：搜索不到数据、查询慢或服务启动失败时，按顺序定位问题。

## 1. 先确认服务可用

```powershell
Invoke-RestMethod http://localhost:9200
Invoke-RestMethod http://localhost:9200/_cluster/health
```

重点查看集群状态：

| 状态 | 含义 |
|---|---|
| `green` | 主分片和副本分片正常 |
| `yellow` | 主分片正常，副本未全部分配；单节点学习环境常见 |
| `red` | 有主分片不可用，应立即排查 |

## 2. 常用检查接口

```text
GET /_cat/indices?v
GET /products/_mapping
GET /products/_count
GET /_nodes/stats/jvm,fs
```

它们分别用于查看索引、Mapping、文档数量、JVM 和磁盘状态。

## 3. 常见现象速查

| 现象 | 常见原因 | 第一动作 |
|---|---|---|
| 容器启动后退出 | Docker 内存不足、配置错误 | 看 `docker compose logs` 第一段异常 |
| Java 连接失败 | 服务未启动、HTTP/HTTPS 或认证配置不匹配 | 先用 REST 请求验证地址 |
| `fielddata` 报错 | 对 `text` 字段做排序或聚合 | 改用 `.keyword` 或调整 Mapping |
| 搜索不到刚写入的数据 | 还未 refresh、索引或字段名写错 | 按 ID 查询并检查 Mapping |
| 查询很慢 | 深分页、通配符查询、大量分片 | 先看请求和慢日志 |
| 磁盘接近满 | 索引增长、保留策略缺失 | 检查索引大小和数据生命周期 |

## 4. 排障顺序

1. 确认 Elasticsearch 进程或容器存活。
2. 确认集群健康、磁盘和 JVM 没有明显异常。
3. 确认索引存在，文档实际已经写入。
4. 确认 Mapping 与查询类型匹配，例如 `text` 用 `match`、`keyword` 用 `term`。
5. 用最小 REST 请求复现，再检查 Java 代码。
6. 最后再处理分页、分片、聚合和性能参数。

## 5. 生产环境最容易遗漏的事

- 启用认证、TLS 和最小权限。
- 规划索引生命周期、备份和恢复演练。
- 监控 JVM 堆、磁盘、线程池拒绝和查询延迟。
- 控制索引数量和分片数量，避免“小索引、小分片”无限增长。
- 让数据库到 Elasticsearch 的同步链路可重试、可补偿、可追踪。

## 6. 学习完成检查

完成下面四件事，说明你已经掌握 Elasticsearch 入门主线：

- 用 Docker 启动单节点服务并执行 REST 请求。
- 创建带 Mapping 的 `products` 索引。
- 写出 `match`、`term`、`range` 与 `bool` 查询。
- Java 或 Spring Boot 执行一次商品搜索。

后续可以继续学习分词器、同义词、Bulk API、索引生命周期、快照恢复、集群部署和向量检索；这些能力都建立在本系列的索引、Mapping 和查询基础之上。
