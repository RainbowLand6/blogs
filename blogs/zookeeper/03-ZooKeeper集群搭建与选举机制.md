---
date: 2026-06-23
is_published: true
title: 03-ZooKeeper集群搭建与选举机制
tags:
  - ZooKeeper
  - 集群
  - 选举
  - ZAB
categories:
  - ZooKeeper
---

## 一、为什么要搭集群？

单机 ZooKeeper 挂了 = 所有依赖它的服务瘫痪。生产环境至少要搭 3 台机器组成的集群，问题变成：

- 单台机器挂了，集群还能正常工作吗？**能，只要过半存活。**
- 数据会不会丢？**不会，写入的数据由过半节点确认后才算提交。**

ZooKeeper 的集群数量口诀：**奇数台，至少 3 台**。常见配置：

| 集群规模 | 可容忍故障数 | 推荐场景 |
|---|---|---|
| 3 台 | 1 台 | 中小型业务 |
| 5 台 | 2 台 | 大型业务 |
| 7 台 | 3 台 | 对可用性极高的场景 |

为什么是奇数？因为 3 台能容忍 1 台故障，4 台也只能容忍 1 台故障，多一台没提升容错能力，还增加了通信开销。

## 二、搭建一个 3 节点集群（实战）

假设你有三台机器或一台机器上的三个端口（练习用）：

| 节点 | IP | 通信端口 | 选举端口 | 客户端端口 |
|---|---|---|---|---|
| server.1 | 192.168.1.101 | 2888 | 3888 | 2181 |
| server.2 | 192.168.1.102 | 2888 | 3888 | 2181 |
| server.3 | 192.168.1.103 | 2888 | 3888 | 2181 |

### 2.1 在每个节点上安装 ZooKeeper

在每台机器上重复第二篇的安装步骤（下载、解压），或者用 scp 复制：

```bash
scp -r /opt/zookeeper root@192.168.1.102:/opt/
scp -r /opt/zookeeper root@192.168.1.103:/opt/
```

### 2.2 编写集群配置文件

每台机器的 `conf/zoo.cfg` 基本一样，只是 myid 不同。

**在你的本机实操（单机模拟集群）：**

如果想在一台机器上跑三个 ZooKeeper 进程来练习，创建三个目录：

```bash
mkdir -p /opt/zk-cluster/{zk1,zk2,zk3}/{data,conf,logs}
```

每个节点的 `zoo.cfg` 配置如下（以 zk1 为例）：

```properties
# zk1/conf/zoo.cfg
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/opt/zk-cluster/zk1/data
clientPort=2181

# 集群配置（关键！）
server.1=127.0.0.1:2888:3888
server.2=127.0.0.1:2889:3889
server.3=127.0.0.1:2890:3890
```

`server.A=B:C:D` 的含义：

| 部分 | 含义 |
|---|---|
| A | 节点的 myid（一个数字，全局唯一） |
| B | 该节点的 IP 地址 |
| C | Follower 与 Leader 的通信端口 |
| D | 选举端口（发生 Leader 选举时使用） |

**zk2 的 zoo.cfg** 除了 `dataDir` 和 `clientPort` 不同，集群配置完全一样：

```properties
dataDir=/opt/zk-cluster/zk2/data
clientPort=2182
server.1=127.0.0.1:2888:3888
server.2=127.0.0.1:2889:3889
server.3=127.0.0.1:2890:3890
```

**zk3 的 zoo.cfg：**

```properties
dataDir=/opt/zk-cluster/zk3/data
clientPort=2183
server.1=127.0.0.1:2888:3888
server.2=127.0.0.1:2889:3889
server.3=127.0.0.1:2890:3890
```

### 2.3 创建 myid 文件

每个节点的 `dataDir` 里必须有一个 `myid` 文件，内容是当前节点的编号：

```bash
echo "1" > /opt/zk-cluster/zk1/data/myid
echo "2" > /opt/zk-cluster/zk2/data/myid
echo "3" > /opt/zk-cluster/zk3/data/myid
```

**成功标志：** 三个文件分别包含 1、2、3。

```bash
cat /opt/zk-cluster/zk1/data/myid  # 输出：1
cat /opt/zk-cluster/zk2/data/myid  # 输出：2
cat /opt/zk-cluster/zk3/data/myid  # 输出：3
```

### 2.4 逐个启动节点

```bash
# 分别启动三个节点
/opt/zookeeper/bin/zkServer.sh start /opt/zk-cluster/zk1/conf/zoo.cfg
/opt/zookeeper/bin/zkServer.sh start /opt/zk-cluster/zk2/conf/zoo.cfg
/opt/zookeeper/bin/zkServer.sh start /opt/zk-cluster/zk3/conf/zoo.cfg
```

### 2.5 验证集群状态

查看每个节点的角色：

```bash
/opt/zookeeper/bin/zkServer.sh status /opt/zk-cluster/zk1/conf/zoo.cfg
# Mode: follower  或  Mode: leader

/opt/zookeeper/bin/zkServer.sh status /opt/zk-cluster/zk2/conf/zoo.cfg
# Mode: follower

/opt/zookeeper/bin/zkServer.sh status /opt/zk-cluster/zk3/conf/zoo.cfg
# Mode: follower
```

三个节点中，**一个 Leader，两个 Follower**。这就是你搭好的第一个 ZooKeeper 集群。

## 三、Leader 选举是怎么工作的？

### 3.1 什么时候会触发选举？

- 集群刚启动时（没有人是 Leader）
- Leader 宕机或网络断开时
- Follower 发现 Leader 失联超过 `syncLimit`

### 3.2 选举的核心规则：ZAB 协议

ZooKeeper 使用的不是 Raft，而是一个叫 **ZAB（ZooKeeper Atomic Broadcast）** 的协议。选举过程很简单：

**每条投票只包含两个东西：**
1. **myid**：节点自己的编号
2. **zxid**：节点处理过的最新事务 ID（越大说明数据越新）

**选举逻辑：**
1. 每个节点投自己一票，然后广播出去
2. 收到别人的投票后比较：**先比 zxid，zxid 大的赢；zxid 相同比 myid，myid 大的赢**
3. 如果某个节点拿到了**过半票数**，它就成为新 Leader

这样说可能有点抽象，来一个具体的例子：

### 3.3 模拟一次选举

三台机器 myid 分别为 1、2、3，刚启动，zxid 都是 0。

```
第一轮投票：
  节点1 投给自己
  节点2 投给自己
  节点3 投给自己

各自广播后：
  节点1 收到自己的票(投1)和节点2的票(投2)、节点3的票(投3)
  zxid 都一样是 0，所以比 myid → 最大的 3 胜出
  节点1 改投节点3

  节点2 同理，也改投节点3
  节点3 也收到票，继续投自己

结果：节点3 获得 3 票（过半，过半 = 2），成为 Leader。
```

如果 Leader（节点3）挂了：

```
节点1 和节点2 发现 Leader 失联，发起新选举
  节点1：myid=1, zxid=15
  节点2：myid=2, zxid=15

  节点1 投给自己，节点2 投给自己
  节点1 比较：zxid 相同(15=15)，myid 2 > 1 → 改投节点2
  节点2 比较：zxid 相同 → 继续投自己

结果：节点2 获得 2 票（过半），成为新 Leader。
```

### 3.4 客户端如何找到 Leader？

ZooKeeper 客户端连接时通常会配置所有集群地址：

```bash
zkCli.sh -server 192.168.1.101:2181,192.168.1.102:2181,192.168.1.103:2181
```

客户端会随机连接其中一个节点。如果连的不是 Leader 想写数据，该节点会自动把写请求转发给 Leader，对客户端透明。

## 四、手动测试 Leader 故障切换

这是一个很有成就感的实验：

**第一步：查看当前 Leader。**

```bash
# 假设发现 zk3 是 Leader（端口 2183）
```

**第二步：杀掉 Leader。**

```bash
# 找到进程并杀掉
ps aux | grep zoo.cfg | grep zk3
kill -9 <zk3的PID>
```

**第三步：检查剩余节点。**

```bash
/opt/zookeeper/bin/zkServer.sh status /opt/zk-cluster/zk1/conf/zoo.cfg
# Mode: leader  ← 可能 zk1 被选为新 Leader

/opt/zookeeper/bin/zkServer.sh status /opt/zk-cluster/zk2/conf/zoo.cfg
# Mode: follower
```

你在几秒钟之内就能看到 Leader 自动切换。**这就是 ZooKeeper 高可用的核心价值。**

**第四步：重启旧 Leader。**

```bash
/opt/zookeeper/bin/zkServer.sh start /opt/zk-cluster/zk3/conf/zoo.cfg
```

重启后 zk3 会成为 Follower，不影响现有集群。

## 五、集群配置的最佳实践

### 5.1 端口规划要规范

| 端口类型 | 推荐值 | 说明 |
|---|---|---|
| clientPort | 2181 | 客户端连接端口 |
| 通信端口 | 2888 | Leader → Follower 数据同步 |
| 选举端口 | 3888 | 选举通信 |

多实例在同一台机器时，给每个实例偏移量，比如：2181/2888/3888、2182/2889/3889、2183/2890/3890。

### 5.2 生产环境注意事项

- **用奇数台机器**：3、5、7，不要用偶数。
- **dataDir 和 dataLogDir 分开**：事务日志（dataLogDir）用独立磁盘，SSD 最佳。
- **各机器时间同步**：`ntpdate` 或 `chronyd`，时间偏差太大会导致集群异常。
- **JVM 堆内存**：一般 4GB-8GB，不要太大（ZooKeeper 数据在内存里，但数据结构简单）。
- **监控 2181 端口**：端口不通 = 客户端连不上。

### 5.3 常见错误排查

| 错误现象 | 可能原因 | 解决 |
|---|---|---|
| `Cannot open channel to X at election address` | myid 与 server 配置不匹配 | 检查 myid 文件和 zoo.cfg 的 server.X |
| 集群只有一个节点 | zoo.cfg 里没写完整集群配置 | 确保所有节点的集群配置一致 |
| `fencing token` 相关错误 | 网络分区 | 检查网络连通性 |
| 反复选举 | 网络延迟过高、时间不同步 | 检查网络和 NTP |

推荐用这种方式快速确认集群是否工作正常：
```bash
# 连上任意节点
zkCli.sh -server 127.0.0.1:2181

# 写一条数据
create /cluster-test "hello-cluster"

# 从另一个节点读
zkCli.sh -server 127.0.0.1:2182
get /cluster-test
# 如果能读到 "hello-cluster"，说明集群数据同步正常
```

## 六、本篇总结

这篇我们完成了：

1. 在一台机器上模拟了一个 3 节点的 ZooKeeper 集群
2. 理解了 myid + zoo.cfg 的集群配置方式
3. 搞懂了 Leader 选举的基本原理（zxid 优先、myid 次之、过半当选）
4. 手动测试了 Leader 挂掉后的自动切换

下一篇，我们深入 ZooKeeper 最核心的数据模型——znode 的四种类型及其使用场景。
