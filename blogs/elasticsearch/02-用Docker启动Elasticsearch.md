---
date: 2026-07-16
is_published: true
title: 用 Docker 启动 Elasticsearch
tags: [Elasticsearch, Docker, 入门]
categories: [Elasticsearch]
---

# 用 Docker 启动 Elasticsearch

目标：启动一个仅供本机学习的单节点 Elasticsearch，并用 REST API 验证服务。

## 1. 创建 `compose.yaml`

新建空目录，创建如下文件：

```yaml
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:9.3.0
    container_name: es-local
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - xpack.security.http.ssl.enabled=false
      - ES_JAVA_OPTS=-Xms512m -Xmx512m
    ports:
      - "9200:9200"
```

这份配置关闭认证和 HTTPS，只为了避免本地入门被证书和账号配置打断。生产环境必须使用 Elasticsearch 默认启用的安全能力，并限制网络访问。

## 2. 启动服务

```bash
docker compose up -d
docker compose ps
```

Elasticsearch 比 Redis、Kafka 更吃内存。若容器频繁退出，先在 Docker Desktop 中把总内存提高到至少 4GB。

查看启动日志：

```bash
docker compose logs --tail=50 elasticsearch
```

## 3. 验证服务

PowerShell：

```powershell
Invoke-RestMethod http://localhost:9200
```

macOS、Linux 或 Git Bash：

```bash
curl http://localhost:9200
```

返回 JSON 中包含集群名、版本号和 `tagline`，说明服务正常。

## 4. 创建第一份文档

PowerShell：

```powershell
Invoke-RestMethod -Method Put `
  -Uri "http://localhost:9200/products/_doc/1001" `
  -ContentType "application/json" `
  -Body '{"name":"无线蓝牙耳机","price":299}'
```

然后查询：

```powershell
Invoke-RestMethod "http://localhost:9200/products/_doc/1001"
```

## 5. 停止服务

```bash
docker compose stop
```

完全删除容器：

```bash
docker compose down
```

下一篇会解释 `products`、`_doc`、字段类型和 Mapping 分别是什么。
