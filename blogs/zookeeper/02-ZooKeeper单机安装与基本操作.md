---
date: 2026-06-23
is_published: true
title: 02-ZooKeeper单机安装与基本操作
tags:
  - ZooKeeper
  - 安装
  - zkCli
categories:
  - ZooKeeper
---

## 一、开始之前：确认环境

本教程以 **Linux（CentOS 7 / Ubuntu 20.04）** 为主，最后也会给 Windows 的简要说明。你需要：

- JDK 8 或更高版本（ZooKeeper 是用 Java 写的）
- 一台能联网的机器（虚拟机也行）
- 基本的 Linux 命令操作能力

**成功标志**：确认 JDK 已安装。

```bash
java -version
# 输出类似：openjdk version "1.8.0_352"
```

如果没装 JDK，先执行：

```bash
# CentOS
yum install -y java-1.8.0-openjdk-devel

# Ubuntu
apt-get install -y openjdk-8-jdk
```

## 二、下载并解压 ZooKeeper

**第一步：进入目录并下载。**

```bash
cd /opt
wget https://archive.apache.org/dist/zookeeper/zookeeper-3.8.1/apache-zookeeper-3.8.1-bin.tar.gz
```

如果 wget 慢或报错，可以自己从 [Apache ZooKeeper 下载页面](https://zookeeper.apache.org/releases.html) 下载后上传到服务器。

**第二步：解压。**

```bash
tar -xzf apache-zookeeper-3.8.1-bin.tar.gz
mv apache-zookeeper-3.8.1-bin zookeeper
```

> 注意：一定要下载 `-bin.tar.gz` 结尾的包，不要下载源码包（`-src` 结尾），否则启动会报找不到主类的错。

**成功标志：** 能看到目录结构。

```bash
ls /opt/zookeeper
# 输出：bin  conf  docs  lib  LICENSE.txt  NOTICE.txt  README.md  README_packaging.md
```

## 三、配置 ZooKeeper

ZooKeeper 的配置文件在 `conf/zoo.cfg`。默认没有这个文件，从模板复制一份：

```bash
cd /opt/zookeeper
cp conf/zoo_sample.cfg conf/zoo.cfg
```

编辑 `conf/zoo.cfg`：

```bash
vim conf/zoo.cfg
```

关键配置项解释：

```properties
# 心跳间隔，单位毫秒。Follower 和 Leader 之间以此频率通信
tickTime=2000

# Follower 在启动时与 Leader 同步数据的超时时间，单位是 tickTime 的倍数
# 即 10 * 2000ms = 20秒
initLimit=10

# Leader 和 Follower 之间的心跳超时
# 即 5 * 2000ms = 10秒
syncLimit=5

# 快照数据的存储目录（重要！生产环境建议用独立磁盘）
dataDir=/opt/zookeeper/data

# 客户端连接端口
clientPort=2181

# 以下两行是自动清理配置（推荐开启）
# 自动清理快照和事务日志，保留最近 3 份，每 8 小时检查一次
autopurge.snapRetainCount=3
autopurge.purgeInterval=8
```

创建数据目录：

```bash
mkdir -p /opt/zookeeper/data
```

**成功标志：** 配置文件已保存，数据目录已创建。

## 四、启动和停止 ZooKeeper

**启动：**

```bash
cd /opt/zookeeper
bin/zkServer.sh start
```

**验证是否启动成功：**

```bash
bin/zkServer.sh status
# 输出：ZooKeeper JMX enabled by default
# Using config: /opt/zookeeper/bin/../conf/zoo.cfg
# Client port found: 2181. Client address: localhost.
# Mode: standalone
```

看到 `Mode: standalone` 就说明单机模式启动成功了。

**查看日志：**

```bash
tail -f /opt/zookeeper/logs/zookeeper--server-*.out
# 或者在 conf/logback.xml 里改了日志目录的话，去对应位置找
```

> 如果启动失败，先看日志。90% 的情况是 dataDir 目录不存在或没有写入权限。

**停止：**

```bash
bin/zkServer.sh stop
```

## 五、使用 zkCli 连接并操作

`zkCli` 是 ZooKeeper 自带的命令行客户端，连接本机：

```bash
bin/zkCli.sh -server 127.0.0.1:2181
```

连接成功后你会看到类似提示：

```
[zk: 127.0.0.1:2181(CONNECTED) 0]
```

接下来我们用它操作 znode（ZooKeeper 的数据节点，类似文件系统中的"文件"或"目录"）。

### 5.1 查看帮助

```bash
help
```

会列出所有可用命令。常用的就几个：`ls`、`create`、`get`、`set`、`delete`、`stat`。

### 5.2 列出节点（ls）

```bash
ls /
# 输出：[zookeeper]

ls /zookeeper
# 输出：[config, quota]
```

ZooKeeper 启动后会自动创建 `/zookeeper` 节点，存放自身的管理信息。

### 5.3 创建节点（create）

```bash
# 创建一个持久节点
create /hello world

# 创建一个带顺序号的节点
create -s /seq/order- 1
# 实际创建的是 /seq/order-0000000000（序号自动追加）

# 创建一个临时节点（当前会话断开后自动删除）
create -e /tmp/session-data alive
```

> 注意：创建 `/seq/order-` 这种路径时，父节点 `/seq` 必须已经存在，否则会报 `KeeperErrorCode = NoNode`。

### 5.4 读取节点数据（get）

```bash
get /hello
# 输出：
# world
# cZxid = 0x2
# ctime = Tue Jun 23 14:30:00 CST 2026
# mZxid = 0x2
# mtime = Tue Jun 23 14:30:00 CST 2026
# pZxid = 0x2
# cversion = 0
# dataVersion = 0
# aclVersion = 0
# ephemeralOwner = 0x0
# dataLength = 5
# numChildren = 0
```

理解这些返回信息：

| 字段 | 含义 |
|---|---|
| cZxid | 创建该节点的事务 ID |
| ctime | 创建时间 |
| mZxid | 最后修改的事务 ID |
| mtime | 最后修改时间 |
| dataVersion | 数据版本号（每次 set 加 1） |
| ephemeralOwner | 如果是临时节点，这里是会话 ID；否则为 0x0 |
| dataLength | 数据长度（字节） |
| numChildren | 子节点数量 |

### 5.5 修改节点数据（set）

```bash
set /hello "hello zookeeper"

# 基于版本号的乐观锁更新（dataVersion 匹配才允许更新）
set -v 0 /hello "new value"
# 如果版本号不匹配会报：version No is not valid
```

### 5.6 查看节点元数据（stat）

```bash
stat /hello
# 和 get 输出类似，但不显示数据内容
```

### 5.7 删除节点（delete）

```bash
# 删除节点（节点不能有子节点）
delete /hello

# 递归删除（删除节点及其所有子节点）
deleteall /seq
```

### 5.8 退出

```bash
quit
```

## 六、Windows 上安装 ZooKeeper（简版）

Windows 上也可以跑 ZooKeeper，适合本地开发和练习：

1. 下载 `apache-zookeeper-3.8.1-bin.tar.gz` 并解压（用 7-Zip 或 WinRAR）
2. 复制 `conf/zoo_sample.cfg` 为 `conf/zoo.cfg`
3. 修改 `zoo.cfg` 里的 `dataDir`，例如：`dataDir=D:/zookeeper/data`
4. 创建对应目录
5. 打开 CMD，进入 ZooKeeper 目录，运行：

```cmd
bin\zkServer.cmd
```

连接客户端：

```cmd
bin\zkCli.cmd -server 127.0.0.1:2181
```

操作命令和 Linux 完全一样。

## 七、新手常见踩坑

### 坑1：端口被占用

```bash
# 检查 2181 端口是否被占用
netstat -tlnp | grep 2181
# 或
lsof -i:2181
```

解决方法：改 `clientPort` 为其他端口，或杀掉占用进程。

### 坑2：dataDir 忘创建

错误日志里会有类似 `java.io.FileNotFoundException` 的信息。解决：`mkdir -p /opt/zookeeper/data`。

### 坑3：JDK 版本不对

ZooKeeper 3.8.x 需要 JDK 8+。如果 JDK 版本过低，启动会报错。

### 坑4：防火墙挡了

如果从别的机器连不上 2181 端口：

```bash
# CentOS 7
firewall-cmd --add-port=2181/tcp --permanent
firewall-cmd --reload

# Ubuntu
ufw allow 2181/tcp
```

## 八、本篇总结

到这里你应该已经学会了：

1. 下载、安装、配置 ZooKeeper 单机版
2. 启动和停止 ZooKeeper
3. 用 `zkCli` 进行 znode 的增删改查

下一篇我们将搭建一个真正的 ZooKeeper 集群，并深入理解 Leader 选举是如何工作的。

**趁热打铁建议**：现在就去启动一个 ZooKeeper 单机，把上面的 create / get / set / delete 命令全部敲一遍。敲过的才是你自己的。
