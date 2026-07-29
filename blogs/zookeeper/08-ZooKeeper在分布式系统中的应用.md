---
date: 2026-06-23
is_published: true
title: 08-ZooKeeper在分布式系统中的应用：从配置中心到分布式ID
tags:
  - ZooKeeper
  - 分布式
  - 服务发现
  - 分布式锁
  - 配置中心
categories:
  - ZooKeeper
---

## 一、开篇：ZooKeeper 的六种经典应用模式

前面七篇我们学会了 ZooKeeper 的理论和 Curator 的实战。这一篇我们来一个"俯视图"——看看 ZooKeeper 在实际分布式系统中到底能用在哪。

| 应用模式 | 核心 znode 类型 | 核心机制 |
|---|---|---|
| 配置中心 | PERSISTENT | Watcher + NodeCache |
| 服务发现 | EPHEMERAL | 临时节点 + PathChildrenCache |
| 分布式锁 | EPHEMERAL_SEQUENTIAL | 临时顺序节点 + 序号最小持有 |
| Leader 选举 | EPHEMERAL_SEQUENTIAL | 临时顺序节点 + 序号最小当选 |
| 分布式屏障 | PERSISTENT | 节点存在性作为开关 |
| 分布式 ID 生成 | PERSISTENT_SEQUENTIAL | 顺序节点的自增特性 |

## 二、配置中心

### 原理

把配置数据放在持久节点上，各应用启动时读取，并注册 Watcher 监听变化。

```
/config
├── /config/app-a/
│   ├── db-url    → "jdbc:mysql://..."
│   ├── db-pool   → "20"
│   └── switches  → "{\\"gray\\":true}"
└── /config/app-b/
    └── timeout   → "3000"
```

### 实现要点

- **初始化**：应用启动时从 ZK 拉取全量配置
- **热更新**：用 NodeCache 监听配置变更，收到通知后更新本地缓存
- **回滚**：保留上次配置值，新配置异常时回退
- **兜底**：ZK 不可用时用本地缓存的最后一份有效配置

### 什么场景不适合用 ZK 做配置中心？

- 配置量非常大（几千个 key 以上）——etcd 或 Apollo 可能更合适
- 需要配置版本管理和灰度发布能力——Apollo / Nacos
- 需要多语言 SDK——Apollo / Consul

## 三、服务发现

### 原理

服务提供者启动时在 ZK 上注册一个临时节点，消费者通过 PathChildrenCache 感知变化。

```
/services
├── /services/order-service/
│   ├── /services/order-service/192.168.1.10:8080  (临时节点)
│   └── /services/order-service/192.168.1.11:8080  (临时节点)
└── /services/user-service/
    └── /services/user-service/192.168.1.20:9090   (临时节点)
```

### 完整流程

```
服务端：启动 → create EPHEMERAL 节点 → 维持心跳 → 宕机后节点自动删除

客户端：启动 → getChildren 获取当前可用列表
             → 注册 PathChildrenCache 监听变化
             → 收到 CHILD_ADDED → 添加服务地址到本地列表
             → 收到 CHILD_REMOVED → 从本地列表移除
```

### Curator 实现

服务端注册：
```java
String servicePath = "/services/order-service/" + localIp + ":" + port;
client.create()
      .creatingParentsIfNeeded()
      .withMode(CreateMode.EPHEMERAL)
      .forPath(servicePath, "running".getBytes());
```

客户端发现：
```java
PathChildrenCache cache = new PathChildrenCache(client, "/services/order-service", true);
cache.getListenable().addListener((client, event) -> {
    if (event.getType() == PathChildrenCacheEvent.Type.CHILD_ADDED) {
        String address = event.getData().getPath();
        serviceList.add(address);
        System.out.println("服务上线：" + address);
    } else if (event.getType() == PathChildrenCacheEvent.Type.CHILD_REMOVED) {
        String address = event.getData().getPath();
        serviceList.remove(address);
        System.out.println("服务下线：" + address);
    }
});
cache.start();
```

### Dubbo 和 ZooKeeper

Dubbo 是 ZooKeeper 服务发现最出名的使用者之一。它用 ZK 存储的信息结构大致为：

```
/dubbo
├── /dubbo/com.example.OrderService/
│   ├── /providers/  ← 服务提供者地址列表
│   ├── /consumers/  ← 服务消费者地址列表
│   └── /configurators/  ← 动态配置
```

## 四、分布式锁

### 原理（详细版）

前面第七篇我们用了 `InterProcessMutex`，现在拆开看它底层是怎么工作的。

```
流程：
1. 客户端 A 想获取锁 /locks/mylock
2. 在 /locks/mylock 下创建临时顺序节点：_c_0000000000
3. 获取 /locks/mylock 下所有子节点，排序
4. 如果自己创建的节点序号最小 → 获取锁成功
5. 如果不是最小 → 对自己前一个节点注册 Watcher
6. 前一个节点被删 → Watcher 触发 → 回到步骤 3 检查

   /locks/mylock
   ├── _c_0000000000  ← A 创建的（最小，获取锁）
   ├── _c_0000000001  ← B 创建的（等待，watch _c_0000000000）
   └── _c_0000000002  ← C 创建的（等待，watch _c_0000000001）
```

这个设计有两个精妙之处：
- **公平**：按序号排队，先到先得
- **避免羊群效应**：每个节点只 watch 前一个，不会所有人同时醒来

### 和 Redis 分布式锁的对比

| 特性 | ZooKeeper 锁 | Redis 锁（Redisson） |
|---|---|---|
| 一致性 | 强一致（过半确认） | 最终一致（需用 RedLock 增强） |
| 性能 | 中等 | 高 |
| 自动释放 | 会话断开自动删除临时节点 | 需设置过期时间 + 看门狗续期 |
| 公平性 | 默认支持（顺序节点排队） | 需额外实现 |
| 适合场景 | 一致性要求高的场景 | 性能要求高的场景 |

**选型一句话**：锁的值很值钱（比如金融交易），用 ZK 锁；锁的操作很频繁（每秒上万次），用 Redis 锁。

## 五、Leader 选举

### 原理

和分布式锁几乎一样，只是竞争的不是"唯一的操作权"而是"Leader 身份"。

```
/election
├── /election/candidate-0000000000  ← 节点A创建的（当选Leader）
├── /election/candidate-0000000001  ← 节点B创建的（备用）
└── /election/candidate-0000000002  ← 节点C创建的（备用）
```

序号最小的当选 Leader。Leader 宕机后它的临时节点自动删除，序号次之的候选者自动晋升。

### 典型应用：定时任务互斥执行

假设有 3 台机器都部署了同一个定时任务，但每次只该有一台执行：

```java
LeaderSelector selector = new LeaderSelector(client, "/jobs/clean-db",
        new LeaderSelectorListenerAdapter() {
            @Override
            public void takeLeadership(CuratorFramework client) throws Exception {
                // 当选 Leader 后执行定时任务
                System.out.println("开始清理数据库...");
                cleanExpiredData();

                // 释放 Leader 身份（退出此方法即触发重新选举）
                // 如果想让这台机器一直当 Leader：
                // Thread.sleep(Long.MAX_VALUE);
            }
        });
selector.autoRequeue();  // 释放后自动重新排队
selector.start();
```

## 六、分布式屏障（Barrier）

分布式屏障就像跑步比赛的起跑线——所有选手（计算节点）必须都准备好，才能同时开始。

### 实现方式

用 ZooKeeper 的节点存在性作为屏障开关：

```
/barriers/my-barrier（持久节点，不存在时所有人等待）
```

```java
// 节点A、B、C 都执行这段代码

// 等待屏障节点被创建
while (client.checkExists().forPath("/barriers/my-barrier") == null) {
    Thread.sleep(1000);
    System.out.println("等待屏障...");
}

// 屏障解除，所有节点同时开始
System.out.println("屏障解除，开始执行！");
```

管理员在 zkCli 中执行：
```bash
create /barriers/my-barrier "go"
```

所有等待的节点就会同时继续执行。

## 七、分布式 ID 生成器

利用 PERSISTENT_SEQUENTIAL 节点的自动递增序号：

```
CREATE -s /id-generator/order/order-
# → /id-generator/order/order-0000000001
# → /id-generator/order/order-0000000002
# → /id-generator/order/order-0000000003
```

```java
String path = client.create()
        .withMode(CreateMode.PERSISTENT_SEQUENTIAL)
        .forPath("/id-generator/order/order-");

// 提取序号
String id = path.substring(path.lastIndexOf('-') + 1);
System.out.println("生成的 ID：" + id);
```

**优点**：全局唯一、顺序递增、实现简单。

**缺点**：性能有限（每次生成都要走 ZK 写操作），不适合极高并发场景。高并发场景更适合雪花算法（Snowflake）。

## 八、ZK 能做到和不能做到的事

### 适合做的事（你的最佳武器）

- 选主、加锁、同步——分布式协调的原生需求
- 少量配置的热更新——不超过几百个 key
- 服务注册与发现——配合临时节点很自然
- 集群元数据管理——Kafka、HBase 都是这样用

### 不适合做的事（别踩坑）

- 大容量存储——zk 不是数据库，znode 最大 1MB，全在内存
- 高吞吐消息队列——写操作太贵，扛不住高频写入
- 海量配置项——几千个 key 以上会有性能问题
- 跨数据中心——ZooKeeper 的 ZAB 协议对延迟敏感

## 九、总结

这一篇我们看到了 ZooKeeper 在真实分布式系统中的六种经典玩法。本质上它们都可以归结为三类模式：

1. **数据存储与监听**：配置中心、服务发现
2. **顺序节点竞争**：分布式锁、Leader 选举、分布式 ID
3. **存在性开关**：分布式屏障、集群成员管理

下一篇（最后一篇），我们聊聊 ZooKeeper 上线后容易遇到的问题和排查方法。
