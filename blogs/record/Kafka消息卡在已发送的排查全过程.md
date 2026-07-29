---
title: Kafka消息卡在"已发送"的排查全过程
date: 2026-06-18
tags: [Kafka, 消息队列, 故障排查, 运维]
categories: 技术排查
---

## 问题现象

测试环境业务消息状态表中，大量消息长时间停留在"已发送"状态，重发次数为 0，一直没有被标记为"已消费"。涉及的 topic `erp.order.insert` 。

## 排查过程

### 第一步：检查消费服务进程

先确认消费方服务是否存活：

```bash
netstat -tulnp | grep java
```

erp 等相关服务的端口都在监听，进程状态正常，不是服务挂了。

### 第二步：检查 Kafka broker 连接

通过启动脚本找到 Kafka broker 地址 `192.168.x.140:9092`，尝试列出消费组：

```bash
/usr/local/kafka/kafka_2.12-3.4.0/bin/kafka-consumer-groups.sh \
  --bootstrap-server 192.168.x.140:9092 --list
```

结果报错：**Connection to node -1 could not be established. Broker may not be available.** — broker 连不上。

### 第三步：检查 Kafka 进程

```bash
ps -ef | grep -i kafka | grep -v grep
```

关键发现：只有 `QuorumPeerMain`（Zookeeper）在运行，**没有 `kafka.Kafka` 主类的进程**。Kafka broker 挂了，ZK 还活着。

### 第四步：启动 Kafka，重启消费服务

```bash
# 启动 Kafka
nohup /usr/local/kafka/kafka_2.12-3.4.0/bin/kafka-server-start.sh \
  /usr/local/kafka/kafka_2.12-3.4.0/config/server.properties &

# 重启 erp 服务后，查看消费组状态
/usr/local/kafka/kafka_2.12-3.4.0/bin/kafka-consumer-groups.sh \
  --bootstrap-server 192.168.x.140:9092 --describe --group test23-erp
```

消费组恢复正常，CONSUMER-ID 有值，所有 topic 的 LAG = 0，消费者 `192.168.x.23` 已连接成功。

## 根因分析

经过反复验证，最终还原了整个故障链路：

```
Kafka broker 挂掉
  ↓
生产端尝试投递消息 → 投递失败 → DB 状态标记为"已发送"（假状态）
  ↓
Kafka broker 恢复 + 消费服务重启
  ↓
新消息正常走通：待发送 → 发送到 Kafka → 已发送 → 消费者消费 → 已消费
  ↓
旧消息卡在假"已发送" —— Kafka 里根本没有它们，消费者永远不会消费
```

**核心矛盾**：消息表状态和 Kafka 实际状态不一致。生产端把"发送"和"写入状态表"放在了同一个事务/操作中，写入状态成功了，但投递到 Kafka 失败了，导致数据库认为已发送，Kafka 却一无所知。系统没有对这部分假"已发送"消息做兜底重投。

## 解决方案

### 短期止血

将 Kafka 挂掉期间的假"已发送"消息状态改回"待发送"，让发送器重新拾取投递：

```sql
-- 确认影响的记录数
SELECT COUNT(*) FROM message_queue
WHERE status = '已发送'
  AND retry_count = 0
  AND create_time < '2026-06-18 16:40:00';  -- Kafka 恢复前的时间

-- 回退状态
UPDATE message_queue
SET status = '待发送'
WHERE status = '已发送'
  AND retry_count = 0
  AND create_time < '2026-06-18 16:40:00';
```

### 长期优化

1. **Kafka 进程监控与自启动**：将 Kafka 设为 systemd 托管，确保宕机后自动拉起
2. **增加兜底扫描机制**：定时扫描"已发送"超时未消费的消息，触发重投
3. **发送状态与 Kafka 投递解耦**：投递到 Kafka 成功后再写"已发送"，失败写入失败标记，不乱标状态

## 关键知识点

| 概念 | 说明 |
|---|---|
| `kafka-consumer-groups.sh --describe` 中的 LAG | LOG-END-OFFSET - CURRENT-OFFSET，堆积消息数 |
| CONSUMER-ID 为 `-` | 该消费组没有活跃消费者，常见于客户端掉线 |
| LAG=0 + CONSUMER-ID=`-` | 当前无消息进入 + 消费者离线（历史消息已全部消费） |
| 消息"已发送" ≠ 消息在 Kafka 里 | 要区分 DB 状态和实际投递状态，二者可能不一致 |

## 经验教训

1. **排查 Kafka 问题先看 broker 是否存活**，很多时候问题不在消费者，而在于 broker 挂了导致整个链路断裂
2. **"已发送"不等于消息到达了 broker**，更要区分新老消息——老消息可能是中间件宕机期间的"假状态"
3. **消息系统必须有兜底**：扫描 + 重投机制是消息可靠性不可或缺的一环，不能只依赖消费端的 ack
4. **排查时多设备交叉验证**：DB 状态、Kafka broker、消费服务日志三端比对，才能准确定位问题在哪个环节
