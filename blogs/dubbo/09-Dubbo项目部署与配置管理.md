---
date: 2026-07-16
is_published: true
title: Dubbo 项目部署与配置管理
tags: [Dubbo, Docker, 部署, 配置]
categories: [Dubbo]
---

# Dubbo 项目部署与配置管理

目标：把 Provider、Consumer 和 ZooKeeper 放进可部署的环境，并避免把网络地址和密码写死在代码中。

## 1. 配置应来自环境

`application.yml`：

```yaml
dubbo:
  application:
    name: ${DUBBO_APPLICATION_NAME}
  registry:
    address: ${DUBBO_REGISTRY_ADDRESS}
  protocol:
    port: ${DUBBO_PROTOCOL_PORT:20880}
```

部署时注入：

```text
DUBBO_APPLICATION_NAME=user-provider
DUBBO_REGISTRY_ADDRESS=zookeeper://zookeeper:2181
DUBBO_PROTOCOL_PORT=20880
```

本地运行、测试、生产环境的地址不同，因此不要把 `127.0.0.1:2181` 写死在 Java 代码里。

## 2. Docker Compose 示例

```yaml
services:
  zookeeper:
    image: zookeeper:3.9
    environment:
      ZOO_4LW_COMMANDS_WHITELIST: ruok,stat,mntr

  user-provider:
    image: my-company/user-provider:1.0
    depends_on:
      - zookeeper
    environment:
      DUBBO_APPLICATION_NAME: user-provider
      DUBBO_REGISTRY_ADDRESS: zookeeper://zookeeper:2181
      DUBBO_PROTOCOL_PORT: 20880

  order-consumer:
    image: my-company/order-consumer:1.0
    depends_on:
      - zookeeper
      - user-provider
    environment:
      DUBBO_APPLICATION_NAME: order-consumer
      DUBBO_REGISTRY_ADDRESS: zookeeper://zookeeper:2181
```

容器之间通过服务名 `zookeeper` 通信，不需要向公网开放 ZooKeeper 的 2181 或 Dubbo 的 20880 端口。

## 3. 发布顺序

一个稳妥的发布步骤：

1. 发布 API 契约兼容的新版本。
2. 先部署 Provider，确认服务已注册、健康检查通过。
3. 再部署 Consumer。
4. 观察调用成功率、延迟、超时和错误日志。
5. 发生异常时回滚 Consumer 或 Provider 到上一个兼容版本。

接口升级遵循“先兼容、后迁移、再删除”的方式，避免 Provider 与 Consumer 同时升级失败造成大面积不可用。

## 4. 生产配置重点

- 为 Provider 明确注册 IP、端口和网卡策略，特别是容器、多网卡和云主机环境。
- 通过统一配置平台或部署变量管理注册中心地址、超时和限流参数。
- 将 ZooKeeper 集群地址、账号密码、证书等敏感信息放在密钥管理系统中。
- 建立服务调用指标：QPS、成功率、P95/P99 延迟、超时数、重试数。

Docker 的镜像和 Compose 基础操作可参考 [[../docker/README]]。下一篇提供 Dubbo 排障时的检查顺序。
