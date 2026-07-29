---
date: 2026-07-16
is_published: true
title: Dubbo 常见问题与排障清单
tags: [Dubbo, ZooKeeper, 排障]
categories: [Dubbo]
---

# Dubbo 常见问题与排障清单

目标：Consumer 无法调用 Provider、调用超时或版本不匹配时，能按依赖链定位问题。

## 1. 先画出依赖链

```text
Consumer -> ZooKeeper -> Provider 注册信息 -> Provider 网络地址 -> Provider 业务实现
```

任何一段不通都会表现为“Dubbo 调用失败”。不要一开始就修改重试次数。

## 2. 常见现象速查

| 现象 | 常见原因 | 第一动作 |
|---|---|---|
| 启动时报“找不到 Provider” | Provider 未启动、未注册、接口不匹配 | 查看 Provider 注册日志和 ZooKeeper 状态 |
| `No provider available` | Consumer 订阅列表为空 | 检查注册中心地址、服务接口、group/version |
| 调用超时 | Provider 慢、网络不通、线程池拥塞 | 看 Consumer 超时日志和 Provider 耗时 |
| 连接被拒绝 | 注册了错误 IP/端口、端口未监听 | 在 Consumer 所在机器测试 Provider 地址 |
| 反序列化失败 | 两端 DTO 或依赖版本不兼容 | 对比 API 包版本和字段类型 |
| 同一写操作执行多次 | 超时后重试、消息重复 | 增加幂等键和数据库唯一约束 |

## 3. 必查日志

Provider：

- 是否成功导出服务。
- 是否成功连接和注册 ZooKeeper。
- 收到请求时的耗时、异常和线程池情况。

Consumer：

- 是否订阅到 Provider 地址。
- 实际选择了哪个 Provider。
- 超时配置、重试次数和根异常。

ZooKeeper：

- 服务节点是否存在。
- 会话是否频繁断开。
- 集群是否健康。

## 4. 排障顺序

1. ZooKeeper 是否存活、Consumer 和 Provider 是否都能连接。
2. Provider 是否成功导出并注册接口。
3. Consumer 是否使用相同的接口全限定名、group、version。
4. Provider 注册的 IP 和端口是否能从 Consumer 所在网络访问。
5. 单次调用耗时是否超过超时设置。
6. 最后再分析负载均衡、重试、线程池和序列化。

## 5. 学习完成检查

完成以下事情，说明你已掌握 Dubbo 入门主线：

- 启动 ZooKeeper 并让 Provider 注册服务。
- Consumer 使用 `@DubboReference` 成功调用 Provider。
- 启动两个 Provider 并理解 Consumer 如何选择实例。
- 解释为什么写操作不能盲目重试。
- 通过日志和注册中心定位一次调用失败。

后续可以继续学习 Dubbo Admin、服务分组与版本、Mock、泛化调用、流量治理、TLS 和多注册中心。先让接口契约、注册发现、超时和幂等性稳定下来，再逐步增加治理能力。
