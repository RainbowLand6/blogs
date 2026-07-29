---
date: 2026-06-23
is_published: true
title: 05-ZooKeeper Watcher机制：当数据变化时自动通知你
tags:
  - ZooKeeper
  - Watcher
  - 事件通知
categories:
  - ZooKeeper
---

## 一、Watcher 是什么？

假设你有一个 Java 应用从 ZooKeeper 读配置。如果配置被管理员改了，你的应用怎么知道？有两种方案：

- **轮询**：每隔 5 秒读一次。缺点是延迟高、无意义请求多。
- **Watcher**：对 znode 注册一个监听器，数据一变 ZooKeeper 就通知你。

Watcher 就是 ZooKeeper 的**事件通知机制**。它不是推送给你的所有客户端，而是你主动说"我对这个节点感兴趣"，ZooKeeper 在变化发生时通知你一次。

## 二、Watcher 的两个关键特性（非常重要！）

### 特性1：一次性触发（one-time trigger）

Watcher 被触发一次后**自动失效**。想继续监听？需要在收到通知后重新注册。

```
你注册 Watcher → znode 变了 → 你收到通知（Watcher 失效）
                                   ↓
                              你想继续监听？
                              是 → 重新注册 Watcher
                              否 → 结束
```

设计成"一次性"有两个原因：
1. **简单**：ZooKeeper 不需要维护长久的 Watcher 状态映射
2. **保序**：你处理通知 → 重新注册 → 继续接收新通知，这个流程保证了事件不会被漏掉

### 特性2：先注册后通知

Watcher 的流程永远是：

```
Client 向 Server 注册 Watcher → Server 返回 ACK → 你才能确定 Watcher 已生效
                                             ↓
                              之后 znode 变化 → Server 通知 Client
```

如果在注册 Watcher 和处理 ACK 之间 znode 变了，通知会排队，你一定能收到。

## 三、Watcher 支持的三种事件类型

| 事件类型 | 触发条件 | 相关命令 |
|---|---|---|
| `NodeCreated` | znode 被创建 | `stat`(不存在节点)、`getData`(不存在节点) |
| `NodeDeleted` | znode 被删除 | `getData`、`exists` |
| `NodeDataChanged` | znode 的数据被修改 | `getData`、`exists` |
| `NodeChildrenChanged` | znode 的子节点列表发生变化（增/删子节点） | `getChildren` |

注意几个容易被忽略的细节：
- `NodeChildrenChanged` 只告诉你"子节点列表变了"，不告诉你是哪个子节点变了
- 子节点自身的**数据变化**不会触发父节点的 `NodeChildrenChanged`
- `NodeDataChanged` 不包括创建和删除

## 四、在 zkCli 中体验 Watcher

打开两个终端，都连上 ZooKeeper：

**终端1（监听者）：**

```bash
zkCli.sh -server 127.0.0.1:2181

# 先确保 /config 存在
create /config "old-value"

# 对 /config 注册 Watcher 并读取数据
get -w /config
# 输出：old-value
# Watcher 已注册
```

**终端2（修改者）：**

```bash
zkCli.sh -server 127.0.0.1:2181

# 修改 /config 的数据
set /config "new-value"
```

回到**终端1**，你会立即看到：

```
WATCHER::

WatchedEvent state:SyncConnected type:NodeDataChanged path:/config
```

**再试一下监控子节点变化：**

终端1：

```bash
# 创建父节点
create /services "parent"

# 对 /services 注册子节点变化监听
ls -w /services
```

终端2：

```bash
create /services/order-service "8080"
```

终端1 收到：

```
WATCHER::

WatchedEvent state:SyncConnected type:NodeChildrenChanged path:/services
```

注意收到的通知里**没有告诉我们具体是哪个子节点创建的**——想获取详情？需要再次 `ls /services`。

### 验证"一次性"特性

终端2：

```bash
set /config "another-value"
```

回到终端1：**不会有任何通知**！因为上一次的 Watcher 已经失效了。要再次监听，需要重新执行 `get -w /config`。

## 五、Java 代码中的 Watcher

使用原生 ZooKeeper API 手动处理 Watcher 比较啰嗦，这里先看一个例子帮助理解原理，后续会用 Curator 简化：

```java
import org.apache.zookeeper.*;

public class SimpleWatcherDemo {
    public static void main(String[] args) throws Exception {
        ZooKeeper zk = new ZooKeeper("127.0.0.1:2181", 3000, null);

        // 注册 Watcher：方式1——在读取时传入
        zk.getData("/config", new Watcher() {
            @Override
            public void process(WatchedEvent event) {
                System.out.println("收到事件：" + event);
                // 关键：收到通知后，重新注册 Watcher
                try {
                    zk.getData("/config", this, null);
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }, null);

        // 方式2——exists 也可以注册 Watcher
        zk.exists("/config", new Watcher() {
            @Override
            public void process(WatchedEvent event) {
                System.out.println("节点变化：" + event);
            }
        });

        Thread.sleep(60000);  // 等待观察
    }
}
```

核心模式就是这个循环：**注册 Watcher → 收到通知 → 处理业务 → 重新注册**。

## 六、Watcher 注意事项和最佳实践

### 6.1 不要用 Watcher 做"可靠的消息投递"

前面说过 Watcher 是一次性的。如果在你收到通知和重新注册之间又发生了一次变化，**第二次变化可能不会触发通知**（虽然有保障机制，但不能 100% 依赖）。如果要做可靠的变更追踪，应该用**版本号**而不是只依赖 Watcher。

### 6.2 避免 Watcher 雪崩

假设 1000 个客户端同时对同一个 znode 注册了 Watcher，zknode 一变化，1000 个通知同时爆发，对服务器压力很大。

解法：
- 让每个客户端 **watch 不同的子节点**，而不是都 watch 同一个节点
- 控制 Watcher 的客户端数量

### 6.3 网络分区时的 Watcher

当客户端与 ZK 集群之间发生网络分区（断网），客户端会收到 `Disconnected` 事件。重连后**所有之前注册的 Watcher 都会失效**，需要重新注册。

### 6.4 Curator 的 Watcher 增强

如果你用 Curator（下一篇讲），它提供了更好用的监听接口：

- `NodeCache`：监听某个节点的数据变化（自动重新注册）
- `PathChildrenCache`：监听某个节点的子节点列表变化（自动重新注册）
- `TreeCache`：监听某个节点及其所有子节点的变化（自动重新注册）

这些工具解决了"手动重新注册"和"只知子节点变化不知具体变化"的问题。

## 七、Watcher 事件的状态类型

查看 Watcher 事件时，你会发现有两个维度：`KeeperState` 和 `EventType`：

| KeeperState | 含义 |
|---|---|
| `SyncConnected` | 连接正常 |
| `Disconnected` | 连接断开 |
| `Expired` | 会话过期 |
| `AuthFailed` | 认证失败 |

| EventType | 触发条件 |
|---|---|
| `None` | 仅状态变化（连接/断开），无节点事件 |
| `NodeCreated` | 节点创建 |
| `NodeDeleted` | 节点删除 |
| `NodeDataChanged` | 节点数据变化 |
| `NodeChildrenChanged` | 子节点列表变化 |

组合起来：当你收到 `state:Disconnected, type:None`，意思是"连接断了，不是任何节点的事件"；当收到 `state:SyncConnected, type:NodeDataChanged`，意思"正常连接状态，节点数据变了"。

## 八、总结

Watcher 是 ZooKeeper 实现"配置自动下发"、"服务自动发现"、"集群自动管理"的基石。

三个要点记住就行：
1. **一次性触发**——收到通知后要重新注册
2. **先注册后通知**——读数据时同时注册 Watcher，保证不丢失事件
3. **子节点监听不告诉你具体内容**——只知"变了"，要手动 re-read

下一篇，我们聊聊 ZooKeeper 的会话管理和 ACL 权限控制。
