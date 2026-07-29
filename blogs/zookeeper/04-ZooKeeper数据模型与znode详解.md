---
date: 2026-06-23
is_published: true
title: 04-ZooKeeper数据模型与znode详解
tags:
  - ZooKeeper
  - znode
  - 数据模型
categories:
  - ZooKeeper
---

## 一、ZooKeeper 的数据模型长什么样？

ZooKeeper 的数据模型非常直观——就是一个**类似 Unix 文件系统的层次化命名空间**。每个节点叫 **znode**（ZooKeeper data node 的缩写），用 `/` 分隔路径。

```
/
├── /zookeeper          ← ZooKeeper 自动创建的管理节点
│   ├── /zookeeper/config
│   └── /zookeeper/quota
├── /services           ← 服务注册根节点
│   ├── /services/order-service
│   └── /services/user-service
├── /config             ← 配置中心根节点
│   └── /config/db-url
└── /locks              ← 分布式锁根节点
    └── /locks/task-001
```

每个 znode 可以同时包含**数据**和**子节点**，这点和文件系统一样——目录下可以有文件，也可以有子目录。

## 二、znode 的四种类型

ZooKeeper 提供了四种 znode，用"持久/临时"和"普通/顺序"两个维度组合而成：

|  | 普通 | 带顺序号 |
|---|---|---|
| **持久（Persistent）** | PERSISTENT | PERSISTENT_SEQUENTIAL |
| **临时（Ephemeral）** | EPHEMERAL | EPHEMERAL_SEQUENTIAL |

### 2.1 持久节点（PERSISTENT）

**特点**：创建后一直存在，除非主动删除。与创建者的会话无关。

```bash
create /config/db-url "jdbc:mysql://localhost:3306/mydb"
```

**典型场景**：
- 配置信息（数据库连接串、开关值）
- 服务注册目录结构（`/services/order-service`）
- 元数据存储

### 2.2 临时节点（EPHEMERAL）

**特点**：创建它的客户端会话断开（主动断开、超时、宕机）时，节点**自动被删除**。临时节点不能有子节点。

```bash
create -e /services/order-service/192.168.1.101:8080 "running"
```

这个命令在 zkCli 执行后，**一旦你退出 zkCli**，这个节点就自动消失了。你可以验证一下：

```bash
# 终端1：创建临时节点
zkCli.sh -server 127.0.0.1:2181
create -e /test-ephemeral "hello"

# 终端2：查看节点存在
zkCli.sh -server 127.0.0.1:2181
get /test-ephemeral
# 输出：hello

# 终端1：quit 退出
# 终端2：再次查看
get /test-ephemeral
# 报错：Node does not exist
```

**典型场景**：
- **服务注册与发现**：服务启动时注册临时节点，宕机后节点自动删除，其他服务自动感知
- **分布式锁**：持有锁的节点会话断开了，锁自动释放
- **集群成员管理**：节点加入时创建临时节点，离开时自动清除

### 2.3 持久顺序节点（PERSISTENT_SEQUENTIAL）

**特点**：持久节点的基础上，ZooKeeper 会自动在节点名后面追加一个 **10 位递增序号**。

```bash
create -s /tasks/task- "clean-db"
# 实际创建：/tasks/task-0000000000

create -s /tasks/task- "send-email"
# 实际创建：/tasks/task-0000000001

create -s /tasks/task- "generate-report"
# 实际创建：/tasks/task-0000000002
```

序号由父节点维护，全局唯一、单调递增。

**典型场景**：
- **分布式队列**：按序号消费任务
- **事件溯源**：记录操作日志，序号作为全局唯一 ID

### 2.4 临时顺序节点（EPHEMERAL_SEQUENTIAL）

**特点**：临时的 + 带序号的。既有"会话断开自动删除"，又有"自动递增序号"。

```bash
create -e -s /election/candidate- "node-A"
# 实际创建：/election/candidate-0000000000

create -e -s /election/candidate- "node-B"
# 实际创建：/election/candidate-0000000001
```

**典型场景**：
- **Leader 选举**：每个候选者创建临时顺序节点，序号最小的当选（后面第七篇会详细讲）
- **分布式锁**：等锁的节点按序号排队，前面的释放了后面的就知道该自己了

**四种类型速查表：**

| 类型 | 持久性 | 顺序号 | 典型场景 |
|---|---|---|---|
| PERSISTENT | 永久存在 | 无 | 配置、元数据 |
| EPHEMERAL | 会话结束自动删除 | 无 | 服务注册、集群成员 |
| PERSISTENT_SEQUENTIAL | 永久存在 | 自动递增 | 分布式队列、事件溯源 |
| EPHEMERAL_SEQUENTIAL | 会话结束自动删除 | 自动递增 | Leader 选举、分布式锁 |

## 三、Stat 结构：znode 的"身份证"

每次用 `get` 或 `stat` 命令，ZooKeeper 都会返回一个 Stat 结构。它不是 znode 的数据，而是它的**元数据**。

```bash
get /hello
# cZxid = 0x6
# ctime = Mon Jun 23 14:00:00 CST 2026
# mZxid = 0x6
# mtime = Mon Jun 23 14:00:00 CST 2026
# pZxid = 0x6
# cversion = 0
# dataVersion = 0
# aclVersion = 0
# ephemeralOwner = 0x0
# dataLength = 5
# numChildren = 0
```

每个字段的详细解释：

| 字段 | 含义 | 通俗理解 |
|---|---|---|
| czxid | 创建时的事务 ID | "出生证明编号" |
| mzxid | 最后修改数据的事务 ID | "最后一次手术编号" |
| pzxid | 最后修改子节点列表的事务 ID | "最后一次添/删子女的编号" |
| ctime | 创建时间 | "出生时间" |
| mtime | 最后修改数据的时间 | "最后手术时间" |
| dataVersion | 数据版本号（每次 set +1） | "数据被改过多少次" |
| cversion | 子节点列表版本号 | "子女变动过多少次" |
| aclVersion | ACL 版本号 | "权限被改过多少次" |
| ephemeralOwner | 如果是临时节点，存会话 ID；否则 0 | "谁创建的临时节点" |
| dataLength | 数据长度（字节） | "数据有多长" |
| numChildren | 直接子节点数量 | "有多少个子女" |

## 四、版本号与乐观锁

ZooKeeper 没有传统数据库那种行锁，它用**版本号**实现乐观锁。

写法：

```bash
# 安全更新：只有 dataVersion=0 时才允许写入
set -v 0 /hello "new-value"
```

如果中间有别人改过：

```bash
# 如果当前 dataVersion=1，你用 -v 0 去更新，报错：
# version No is not valid! /hello
```

这就是乐观锁的精髓：**先检查版本号，匹配了才写入**。在 Java 代码里：

```java
Stat stat = new Stat();
zk.getData("/hello", null, stat);  // 读取数据和版本号
// ... 业务处理 ...
zk.setData("/hello", newData.getBytes(), stat.getVersion());  // 带上版本号更新
```

如果 `stat.getVersion()` 和 ZooKeeper 里的 `dataVersion` 不匹配，`setData` 会抛出 `KeeperException.BadVersionException`，你的代码就可以重试或报错。

## 五、znode 的注意事项和限制

### 5.1 数据大小限制

每个 znode 的**数据默认最大 1MB**。这不是让你省着点用的问题——如果真存接近 1MB 的数据，集群性能会急剧下降。记住：**znode 存小数据，大文件放别处**（比如 OSS/S3/HDFS）。

### 5.2 节点数量

znode 数量可以很多（百万级没问题），但每个节点都要消耗内存。一个 znode 的开销大约 **200-300 字节**（名字 + Stat + 数据指针）。

### 5.3 临时节点的限制

- 临时节点**不能有子节点**
- 临时节点的生命周期绑定在**会话**上，不是进程上（会话超时时间由 `sessionTimeout` 决定）

### 5.4 路径命名规范

虽然没有强制规定，但行业内有一些约定俗成的命名习惯：

```bash
# 好的命名
/services/payment-service/nodes/192.168.1.101:8080
/config/app/database/url
/locks/user-batch-job
/election/master

# 应该避免的
/createOrder   # 驼峰命名（建议用短横线）
/服务/订单      # 中文路径（虽然支持，但不推荐）
/too/deep/path/that/makes/no/sense/at/all  # 层级太深
```

## 六、动手实验：把四种类型都创建一遍

打开你的 zkCli，依次执行以下命令，观察每种节点的行为：

```bash
# 1. 持久节点
create /myapp "My Application"
get /myapp

# 2. 临时节点（记住退出 zkCli 后它会消失）
create -e /myapp/session "active"
get /myapp/session

# 3. 持久顺序节点
create -s /myapp/log "event-a"
create -s /myapp/log "event-b"
create -s /myapp/log "event-c"
ls /myapp
# 看到三个带递增序号的孩子

# 4. 临时顺序节点
create -e -s /myapp/election node-
ls /myapp
# 退出后 election 下的节点消失
```

## 七、本篇总结

znode 是 ZooKeeper 最核心的概念。这一篇我们搞清楚了：

1. ZooKeeper 数据模型 = 层次化文件系统，每个节点叫 znode
2. 四种 znode 类型：PERSISTENT / EPHEMERAL / PERSISTENT_SEQUENTIAL / EPHEMERAL_SEQUENTIAL
3. Stat 结构里每个字段的含义
4. 版本号乐观锁机制
5. 数据限制与命名规范

下一篇，我们讲 ZooKeeper 最迷人的机制——**Watcher**：当 znode 发生变化时，如何让客户端自动收到通知。
