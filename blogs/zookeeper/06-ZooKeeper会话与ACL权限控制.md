---
date: 2026-06-23
is_published: true
title: 06-ZooKeeper会话管理与ACL权限控制
tags:
  - ZooKeeper
  - 会话
  - ACL
  - 权限
categories:
  - ZooKeeper
---

## 一、会话（Session）是什么？

客户端连接 ZooKeeper 会建立一个**会话**。会话是 ZooKeeper 客户端和服务端之间的 TCP 长连接，它是临时节点和 Watcher 的生命基础。

一个会话的生命周期：

```
客户端发起连接 → Server 创建 Session → 分配 sessionId
                ↓
        客户端定期发心跳(ping)
                ↓
┌───────────────┴───────────────┐
│  客户端正常断开（close）        │  心跳超时（sessionTimeout）
│  → 主动销毁 Session             │  → Server 自动销毁 Session
│  → 临时节点删除                 │  → 临时节点删除
└───────────────────────────────┘
```

## 二、会话超时（sessionTimeout）

创建 ZooKeeper 客户端时你需要指定一个 `sessionTimeout` 参数（单位毫秒）。这个参数**非常重要**，它控制：

1. 客户端多久没发心跳，服务端就认定它"死了"
2. 临时节点存活的最长空窗期

```java
// 原生 API
ZooKeeper zk = new ZooKeeper("127.0.0.1:2181", 4000, watcher);
//                                              ^^^^ sessionTimeout=4秒
```

**超时时间的限制：**

- 最小值：通常是 `tickTime * 2`（比如 `2000ms * 2 = 4000ms`）
- 最大值：通常是 `tickTime * 20`（比如 `2000ms * 20 = 40000ms`）
- 最终的实际值由 Server 决定（Server 会返回它可以接受的值）

如果你的 `sessionTimeout` 设得太小，Server 会自动调大；设得太大，Server 会自动调小。

**生产环境建议：10-30 秒。**太短了网络抖动容易超时，太长了（比如几分钟）故障转移太慢。

## 三、连接状态机

ZooKeeper 客户端内部有一个状态机，以下是常见的状态流转：

```
初始 → CONNECTING（正在连）
    → CONNECTED（连上了，正常状态）
    → 如果网络断了 → DISCONNECTED（断连中，Session 还没过期）
        → 重连成功 → CONNECTED
        → 超过 sessionTimeout → EXPIRED（Session 过期）
            → 临时节点全部被删除，Watcher 全部失效
```

**特别注意**：`DISCONNECTED` 和 `EXPIRED` 是完全不同的。

- `DISCONNECTED`：只是网络暂时断了，Session 还在，重连后一切照旧
- `EXPIRED`：Session 已经不存在了，临时节点全丢了，必须重新创建 ZooKeeper 客户端实例

## 四、ACL（访问控制列表）

ZooKeeper 的 ACL 控制"谁可以对某个 znode 做什么"。它和 Linux 文件的 rwx 权限模型不同，用的是**权限位 + 认证方案**的组合。

### 4.1 ACL 的五个组成部分

每个 znode 可以有多个 ACL 条目，每个条目由两部分组成：

```
ACL = (scheme:expression, permissions)
      └── 认证方案 ──┘  └── 权限位 ──┘
```

**五种权限位：**

| 权限位 | 缩写 | 含义 |
|---|---|---|
| CREATE | c | 可以创建子节点 |
| READ | r | 可以读取数据、列出子节点 |
| WRITE | w | 可以修改数据 |
| DELETE | d | 可以删除子节点 |
| ADMIN | a | 可以修改 ACL |

### 4.2 四种内置认证方案

**1. world（默认方案）**

```bash
# 查看 /hello 的 ACL（默认）
getAcl /hello
# 输出：'world,'anyone : cdrwa
# 含义：全世界任何人都可以对这个节点做任何操作
```

`'world,'anyone` 就是字面意思——"任何人在这个世界上"。

**2. auth（已认证用户方案）**

先认证，再赋权：

```bash
# 添加认证用户
addauth digest user1:password123

# 创建一个仅对 user1 可操作的节点
create /secure "secret-data"
setAcl /secure auth:user1:password123:cdrwa
```

**3. digest（用户名密码哈希方案）**

```bash
# 创建节点时直接设置 ACL（使用 SHA1 哈希后的密码）
create /db-config "jdbc-url"
setAcl /db-config digest:admin:QWhFauX+wdrPb5p6oOz+oILaRU8=:cdrwa
```

> `QWhFauX+wdrPb5p6oOz+oILaRU8=` 是 `admin:password123` 的 SHA1 哈希。可以用 ZooKeeper 自带的 `DigestAuthenticationProvider` 生成。

**4. ip（IP白名单方案）**

```bash
# 只允许指定 IP 的客户端操作
create /internal-config "internal"
setAcl /internal-config ip:192.168.1.0/24:cdrwa
```

### 4.3 zkCli 操作 ACL

```bash
# 查看 ACL
getAcl /hello

# 设置 ACL（这会覆盖已有的 ACL）
setAcl /hello world:anyone:r
# 现在 /hello 变成只读了

# 测试权限
set /hello "try-write"
# 报错：Authentication is not valid

# 给只读节点添加可写权限
setAcl /hello world:anyone:rw
set /hello "works-now"
# 成功
```

### 4.4 超级管理员（superDigest）

生产环境可以在启动 ZooKeeper 时配置超级管理员：

```bash
# zoo.cfg 中开启，用你自己生成的加密密码
-Dzookeeper.DigestAuthenticationProvider.superDigest=super:你的加密密码
```

或在启动脚本中：

```bash
export SERVER_JVMFLAGS="-Dzookeeper.DigestAuthenticationProvider.superDigest=super:QWhFauX+wdrPb5p6oOz+oILaRU8="
```

超级管理员可以绕过所有 ACL。

## 五、什么时候需要配置 ACL？

大部分使用 ZooKeeper 的场景不需要配置 ACL，因为：

- ZooKeeper 通常运行在**内网**
- 靠防火墙和网络隔离就能保证安全

但以下场景建议加上 ACL：
- ZooKeeper 上存储了敏感数据（密码、密钥）
- 多个团队共用同一套 ZooKeeper 集群
- 开放了对公网的连接（**强烈不建议**）

## 六、会话管理的常见坑

### 坑1：sessionTimeout 设得太短

如果客户端的业务逻辑处理时间超过了 sessionTimeout，且在那段时间没发心跳，Server 就会认为客户端死了——临时节点被删除，锁被释放，造成严重问题。

**解决**：sessionTimeout 要大于最长业务处理时间，同时把耗时逻辑放到 worker 线程而不是 ZooKeeper 回调线程里。

### 坑2：session 过期后没重建客户端

收到 `Expired` 事件后，旧的 ZooKeeper 客户端实例已经不能用。必须 `close()` 旧实例，`new` 一个新实例。

### 坑3：用同一套 ACL 管理所有节点

不同节点通常需要不同的权限。比如：
- `/config` 下：运维有写权限，应用只有读权限
- `/services` 下：所有服务都可以读写自己的临时注册节点
- `/locks` 下：所有服务都可以创建锁节点

## 七、总结

这一篇我们搞清楚了：

1. 会话是客户端和 ZK Server 的长连接，临时节点的生命周期绑在上面
2. sessionTimeout 要合理设置（10-30秒），太短不稳定，太长故障恢复慢
3. 会话说断连（DISCONNECTED）不等于过期（EXPIRED），过期后必须重建客户端
4. ACL 用 `scheme:expression, permissions` 格式，默认是 `world:anyone:cdrwa`
5. 大部分场景网络安全隔离就够了，敏感数据才上 ACL

下一篇就是期待已久的实战——用 Curator 框架写 Java 代码操作 ZooKeeper！
