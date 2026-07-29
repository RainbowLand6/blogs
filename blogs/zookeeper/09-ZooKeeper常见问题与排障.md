---
date: 2026-06-23
is_published: true
title: 09-ZooKeeper常见问题与排障手册
tags:
  - ZooKeeper
  - 排障
  - 运维
  - 性能
categories:
  - ZooKeeper
---

## 一、开篇：出了问题先看哪里？

ZooKeeper 出问题时别慌，按这个顺序排查：

```
1. 看状态 → zkServer.sh status
2. 看日志 → logs/zookeeper--server-*.out
3. 看端口 → netstat -tlnp | grep 2181
4. 看内存 → jstat -gcutil <PID> 1000
5. 看磁盘 → df -h
```

## 二、常见 KeeperException 错误码速查

当 Java 客户端报 `KeeperException` 时，首先要看错误码：

| 错误码 | 异常类 | 含义 | 常见原因 |
|---|---|---|---|
| ConnectionLoss | ConnectionLossException | 连接丢失 | 网络超时、ZK 挂了、防火墙 |
| SessionExpired | SessionExpiredException | 会话过期 | 长时间断连超过了 sessionTimeout |
| NoNode | NoNodeException | 节点不存在 | 路径写错、临时节点被删了 |
| NodeExists | NodeExistsException | 节点已存在 | 重复创建同名节点 |
| BadVersion | BadVersionException | 版本号不匹配 | 乐观锁冲突，数据被其他客户端改过 |
| NotEmpty | NotEmptyException | 节点有子节点 | 删除非空节点没用递归方式 |
| NoAuth | NoAuthException | 没有 ACL 权限 | 没配置 digest 认证或权限不够 |
| OperationTimeout | OperationTimeoutException | 操作超时 | 操作耗时超过了客户端 timeout |

### 排障思路

**遇到 ConnectionLoss：**

```bash
# 1. 检查 ZK 进程是否活着
ps aux | grep zookeeper

# 2. 检查端口是否在监听
netstat -tlnp | grep 2181

# 3. 检查网络通不通
telnet <zk-host> 2181

# 4. 检查防火墙
iptables -L -n | grep 2181
```

**遇到 SessionExpired：**

收到此异常后，**旧的 ZooKeeper 客户端实例必须丢弃**，重新 `new` 一个：

```java
try {
    client.getData().forPath("/some/path");
} catch (SessionExpiredException e) {
    // 必须新建客户端
    client.close();
    client = createNewClient();
}
```

**遇到 BadVersion：**

说明有并发写入冲突，把读取到的 Stat 版本号带过去更新：

```java
Stat stat = new Stat();
byte[] data = client.getData().storingStatIn(stat).forPath("/data");
// 处理 data...
try {
    client.setData().withVersion(stat.getVersion()).forPath("/data", newData);
} catch (BadVersionException e) {
    // 版本冲突，重试
}
```

## 三、性能问题排查

### 3.1 znode 数量过多

**症状**：集群响应慢、内存高、Leader 选举时间长。

**检查**：

```bash
# 在 zkCli 中执行，查看节点总数（慎用，节点太多可能很慢）
echo stat | nc 127.0.0.1 2181 | grep "Znode count"

# 或 JMX 查看
```

**解决**：
- 清理无用节点（未使用的临时节点残留、过期数据）
- 不要用 znode 存大量数据，大文件放 OSS/HDFS
- 如果 znode 数量确实很大（百万级），考虑增加 ZooKeeper 实例的堆内存

### 3.2 Watcher 过多

**症状**：单次 znode 变更导致延迟峰值。

**检查**：

```bash
echo wchs | nc 127.0.0.1 2181
# 输出示例：
# 342 connections watching 87 paths
# Total watches: 1574
```

如果 `Total watches` 上万，就需要注意了。

**解决**：
- 检查是否有客户端在循环注册 Watcher 但没有正确处理（收到通知后没重新注册）
- 用 TreeCache 代替多个 NodeCache（减少 Watcher 数量）
- 控制客户端数量

### 3.3 写入频率过高

**症状**：写操作延迟大、集群负载高。

ZooKeeper 是写吞吐量有限的系统（通常几百到几千 TPS），因为每次写都要过半确认。

**检查**：

```bash
echo srvr | nc 127.0.0.1 2181
# 查看 Outstanding requests（排队中的请求）
```

**解决**：
- 批量写入：能合并的写入合并后再写
- 如果频繁写临时节点用于心跳，考虑用更轻量级的方案
- 评估是否可以把某些高频写入移到 Redis

### 3.4 磁盘满了

**症状**：ZooKeeper 无法写入快照和事务日志，集群异常。

**检查**：

```bash
# 查看 dataDir 的磁盘使用
df -h /opt/zookeeper/data
du -sh /opt/zookeeper/data/version-2/
```

**解决**：

```properties
# zoo.cfg 中开启自动清理
autopurge.snapRetainCount=3
autopurge.purgeInterval=8
```

如果磁盘已经满了：
```bash
# 停止 ZK，手动清理旧快照（保留最近的 3 个）
cd /opt/zookeeper/data/version-2
ls -t snapshot.* | tail -n +4 | xargs rm -f
# 重启 ZK
```

## 四、常见网络问题

### 4.1 "Cannot open channel to X at election address"

ZooKeeper 节点之间无法通信。最常见的三个原因：

1. **防火墙**挡了 2888（通信端口）和 3888（选举端口）
2. **myid 配置不对**：`dataDir/myid` 的内容与 `zoo.cfg` 中 `server.X` 不匹配
3. **bind 地址配置错误**：多网卡环境需要指定正确的 IP

```bash
# 检查 zoo.cfg 中的 server 配置
grep "server\." /opt/zookeeper/conf/zoo.cfg
# 输出你看到的 IP 端口是否一致

# 验证端口是否监听
ss -tlnp | grep -E "2888|3888"
```

### 4.2 集群频繁切换 Leader

**症状**：日志中反复出现 "LOOKING" 和 "LEADING" / "FOLLOWING"。

**常见原因**：
1. 网络延迟高或不稳定——ZAB 协议对网络要求高
2. JVM GC 停顿时间过长——Full GC 可能达到秒级，触发 Leader 失联
3. 集群节点的系统时间差距太大

**排查**：

```bash
# 检查系统时间同步
ntpstat
# 或
chronyc tracking

# 查看 GC 情况
jstat -gcutil <ZK_PID> 1000
# 关注 FGC（Full GC 次数）和 FGCT（Full GC 耗时）

# 节点间网络延迟
ping <other-zk-node>
```

**解决**：
- 配置 NTP 时间同步
- 调整 JVM 参数：`-Xms4g -Xmx4g -XX:+UseG1GC -XX:MaxGCPauseMillis=200`
- 检查网络设备（交换机、网卡）

## 五、JVM 与内存问题

### 推荐 JVM 参数

```bash
# 生产环境建议配置
JAVA_OPTS="-Xms4g -Xmx4g \
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=200 \
  -XX:ParallelGCThreads=4 \
  -XX:+PrintGCDetails \
  -XX:+PrintGCDateStamps \
  -Xloggc:/opt/zookeeper/logs/gc.log \
  -XX:+UseGCLogFileRotation \
  -XX:NumberOfGCLogFiles=10 \
  -XX:GCLogFileSize=50M"
```

关键是 `-Xmx` 不要设太大（ZooKeeper 数据结构简单，4-8GB 足够）、`MaxGCPauseMillis=200` 避免 GC 时间过长导致 Leader 假死。

### 内存泄漏检测

```bash
jstat -gcutil <PID> 1000 30
# 观察 OU（老年代使用率）是否持续增长不回收
```

如果老年代持续增长，可能是 Watcher/Cache 没有正确关闭。

## 六、四字命令速查

ZooKeeper 自带的"四字命令"（4-letter words）是运维排查的利器：

```bash
echo <命令> | nc 127.0.0.1 2181
```

| 命令 | 功能 | 何时用 |
|---|---|---|
| `stat` | 服务器状态 + 客户端连接数 | 日常巡检 |
| `srvr` | 完整的服务器状态（含版本） | 深入了解运行状况 |
| `cons` | 所有客户端连接详情 | 排查连接数异常 |
| `wchs` | Watcher 汇总 | 排查 Watcher 过多 |
| `wchc` | Watcher 按客户端分组 | 定位哪个客户端 Watcher 多 |
| `wchp` | Watcher 按路径分组 | 定位哪个 znode Watcher 多 |
| `mntr` | 监控指标（键值对格式，适合采集） | 接入监控系统时使用 |
| `dump` | 未完成的会话 + 临时节点 | 排查会话异常 |

> 如果四字命令没反应，检查 `zoo.cfg` 中是否配置了 `4lw.commands.whitelist=*`（新版本默认只开放了部分命令）。

## 七、关键监控指标

接入 Prometheus / Zabbix / Grafana 的话，重点监控这些指标：

| 指标 | 含义 | 告警阈值 |
|---|---|---|
| `zk_avg_latency` | 平均请求延迟 | > 100ms |
| `zk_max_latency` | 最大请求延迟 | > 500ms |
| `zk_outstanding_requests` | 排队中的请求数 | > 50 |
| `zk_znode_count` | znode 总数 | 根据基线设定 |
| `zk_watch_count` | Watcher 总数 | > 10000 |
| `zk_num_alive_connections` | 活跃连接数 | 异常突变 |
| `zk_open_file_descriptor_count` | 打开文件数 | 接近系统限制 |
| 磁盘使用率 | dataDir 磁盘使用 | > 80% |

## 八、故障场景速查表

| 故障现象 | 最可能的原因 | 快速排查命令 |
|---|---|---|
| 客户端连不上 | ZK 进程挂了 / 端口被挡 | `ps aux \| grep zookeeper`, `telnet` |
| 写入失败 | 连的不是 Leader / 磁盘满 | `zkServer.sh status`, `df -h` |
| 读取非常慢 | znode 太多 / 网络延迟高 | `echo stat \| nc 127.0.0.1 2181` |
| 频繁重选举 | 网络抖动 / GC 过长 / NTP 不同步 | `jstat -gcutil`, `ntpstat` |
| 客户端 ConnectionLoss | sessionTimeout 太短 / 网络差 | 检查 sessionTimeout 配置和网络 |
| 磁盘暴涨 | 没开 autopurge | `du -sh dataDir`, 手动清理 |
| 内存耗尽 | 节点数太多 / -Xmx 太小 | `jstat -gcutil`, 增加 -Xmx |

## 九、写在最后

恭喜你读完了这套 ZooKeeper 教程。我们走完了从"ZooKeeper 是什么"到"生产环境排障"的完整旅程。

回顾九篇文章的路线图：

```
概念入门(01) → 单机安装(02) → 集群搭建(03)
                                   ↓
                     数据模型(04) → Watcher(05) → 会话与ACL(06)
                                                     ↓
                                          Curator实战(07) → 经典应用(08)
                                                               ↓
                                                        排障手册(09)
```

如果你看到了这里，建议你做三件事：

1. **亲手搭一个集群**：在一台机器上按第三篇的步骤搭一个 3 节点的集群
2. **跑一遍 Curator 代码**：把第七篇的配置中心例子跑起来，改动配置看热更新效果
3. **搞崩它再修好**：杀掉 Leader 看自动切换、磁盘塞满看报错——只有亲手修过的问题才真正属于你

ZooKeeper 不是一个学一次就能扔下的框架。把它装在你的工具库里，遇到分布式协调问题时，你会感谢自己今天花了时间。
