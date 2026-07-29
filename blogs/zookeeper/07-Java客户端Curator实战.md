---
date: 2026-06-23
is_published: true
title: 07-Java客户端Curator实战：像写单机程序一样写分布式程序
tags:
  - ZooKeeper
  - Curator
  - Java
  - 分布式锁
  - Leader选举
categories:
  - ZooKeeper
---

## 一、为什么要用 Curator 而不是原生 API？

ZooKeeper 的原生 Java API 有几个让大家头疼的问题：

- Watcher 是一次性的，每次收到通知都要手动重新注册
- 连接断开、Session 过期需要自己处理重连逻辑
- 没有现成的分布式锁、Leader 选举等高级工具
- 异常处理繁琐（`KeeperException.ConnectionLossException` 等）

**Curator** 是 Netflix 开源、后捐给 Apache 的 ZooKeeper 高级客户端框架。它做的事情就是帮你把这些坑填了，让你用 Fluent 风格的 API 写干净的代码。

## 二、快速上手：引入依赖并连接

### Maven 依赖

```xml
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-recipes</artifactId>
    <version>5.5.0</version>
</dependency>
```

`curator-recipes` 包含了 `curator-framework` 和 `curator-client`，以及所有高级工具（锁、选举、缓存等）。

### 创建客户端并连接

```java
import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.retry.RetryOneTime;

public class CuratorHelloWorld {
    public static void main(String[] args) throws Exception {
        // 创建一个 Curator 客户端
        CuratorFramework client = CuratorFrameworkFactory.newClient(
                "127.0.0.1:2181",       // 连接地址
                15000,                    // sessionTimeout（毫秒）
                5000,                     // connectionTimeout（毫秒）
                new RetryOneTime(1000)    // 重试策略
        );

        // 启动
        client.start();

        // 接下来操作 ZooKeeper...

        // 关闭
        client.close();
    }
}
```

**重试策略的选择：**

| 策略 | 说明 | 适用场景 |
|---|---|---|
| `RetryOneTime` | 只重试一次 | 快速失败场景 |
| `RetryNTimes(n, sleepMs)` | 重试 n 次，每次间隔 sleepMs | 一般场景 |
| `RetryUntilElapsed(maxElapsedMs, sleepMs)` | 一直重试直到超时 | 要求高可靠性 |
| `ExponentialBackoffRetry(baseSleepMs, maxRetries)` | 指数退避重试 | 生产环境推荐 |

生产环境一般用：

```java
new ExponentialBackoffRetry(1000, 3)
// 首次等1秒，之后2秒、4秒，最多重试3次
```

## 三、CRUD 操作（用 Fluent API）

Curator 的操作链式写法很舒服：

### 创建节点

```java
// 创建持久节点
client.create()
      .forPath("/myapp/config", "jdbc:mysql://localhost/db".getBytes());

// 创建临时节点
client.create()
      .withMode(CreateMode.EPHEMERAL)
      .forPath("/myapp/session", "active".getBytes());

// 创建持久顺序节点
client.create()
      .withMode(CreateMode.PERSISTENT_SEQUENTIAL)
      .forPath("/myapp/logs/log-", "event".getBytes());

// 如果父节点不存在自动创建（很实用！）
client.create()
      .creatingParentsIfNeeded()
      .forPath("/deep/nested/path", "data".getBytes());
```

### 读取数据

```java
byte[] data = client.getData().forPath("/myapp/config");
System.out.println(new String(data));
// 输出：jdbc:mysql://localhost/db

// 同时获取 Stat（版本号等元数据）
Stat stat = new Stat();
byte[] dataWithStat = client.getData().storingStatIn(stat).forPath("/myapp/config");
System.out.println("dataVersion: " + stat.getVersion());
```

### 修改数据

```java
// 普���更新
client.setData().forPath("/myapp/config", "new-value".getBytes());

// 乐观锁更新（dataVersion 匹配才更新）
client.setData()
      .withVersion(stat.getVersion())
      .forPath("/myapp/config", "optimistic-update".getBytes());
// 如果版本不匹配，抛出 BadVersionException
```

### 删除数据

```java
// 删除节点（不能有子节点）
client.delete().forPath("/myapp/config");

// 递归删除
client.delete()
      .deletingChildrenIfNeeded()
      .forPath("/myapp");

// 保证删除（即使连接断开也会重试直到删除）
client.delete()
      .guaranteed()
      .forPath("/important-but-deletable");
```

### 检查是否存在

```java
Stat stat = client.checkExists().forPath("/myapp/config");
if (stat != null) {
    System.out.println("节点存在，版本号：" + stat.getVersion());
}
```

## 四、事件监听（三大 Cache）

Curator 提供了三个 Cache 来替代原生的手动 Watcher 注册。**它们会自动重新注册 Watcher，你不需要再写"收到通知→重新注册"的循环！**

### 4.1 NodeCache：监听单个节点的数据变化

```java
import org.apache.curator.framework.recipes.cache.NodeCache;
import org.apache.curator.framework.recipes.cache.NodeCacheListener;

// 创建 NodeCache，第三个参数表示是否压缩数据
NodeCache nodeCache = new NodeCache(client, "/config/db-url", false);
nodeCache.start(true);  // true = 启动时立即从 ZK 拉一次数据

nodeCache.getListenable().addListener(new NodeCacheListener() {
    @Override
    public void nodeChanged() throws Exception {
        byte[] data = nodeCache.getCurrentData().getData();
        System.out.println("配置更新：" + new String(data));
    }
});

// 使用完后关闭
// nodeCache.close();
```

### 4.2 PathChildrenCache：监听子节点列表变化

```java
import org.apache.curator.framework.recipes.cache.PathChildrenCache;
import org.apache.curator.framework.recipes.cache.PathChildrenCacheEvent;

PathChildrenCache cache = new PathChildrenCache(client, "/services", true);
cache.start();

cache.getListenable().addListener((client, event) -> {
    switch (event.getType()) {
        case CHILD_ADDED:
            System.out.println("子节点加入：" + event.getData().getPath());
            break;
        case CHILD_REMOVED:
            System.out.println("子节点离开：" + event.getData().getPath());
            break;
        case CHILD_UPDATED:
            System.out.println("子节点数据更新：" + event.getData().getPath());
            break;
    }
});
```

### 4.3 TreeCache：监听整个子树的变化

```java
import org.apache.curator.framework.recipes.cache.TreeCache;

// TreeCache = NodeCache + PathChildrenCache，递归监听整棵树
TreeCache treeCache = new TreeCache(client, "/myapp");
treeCache.start();

treeCache.getListenable().addListener((client, event) -> {
    System.out.println("事件类型：" + event.getType());
    System.out.println("节点路径：" + event.getData().getPath());
    if (event.getData().getData() != null) {
        System.out.println("节点数据：" + new String(event.getData().getData()));
    }
});
```

**三大 Cache 对比：**

| Cache | 监听范围 | 使用场景 |
|---|---|---|
| NodeCache | 单个节点数据变化 | 配置监听 |
| PathChildrenCache | 直接子节点增删 | 服务注册与发现 |
| TreeCache | 整个子树（递归） | 配置树监听、复杂拓扑 |

## 五、分布式锁

这是 Curator 最实用、用得最多的功能。

### 5.1 可重入互斥锁（InterProcessMutex）

```java
import org.apache.curator.framework.recipes.locks.InterProcessMutex;

InterProcessMutex lock = new InterProcessMutex(client, "/locks/task-001");

// 加锁（阻塞等待）
lock.acquire();
try {
    // 临界区代码
    System.out.println("拿到锁，执行业务逻辑...");
    Thread.sleep(5000);
} finally {
    // 释放锁
    lock.release();
}
```

这个锁和 Java 的 `ReentrantLock` 用法几乎一样，但它跨 JVM、跨机器！

### 5.2 读写锁

```java
import org.apache.curator.framework.recipes.locks.InterProcessReadWriteLock;

InterProcessReadWriteLock readWriteLock =
        new InterProcessReadWriteLock(client, "/locks/rw-resource");

// 读锁（多个客户端可以同时持有）
InterProcessMutex readLock = readWriteLock.readLock();

// 写锁（只有一个客户端能持有，持有写锁时读锁也不能获取）
InterProcessMutex writeLock = readWriteLock.writeLock();
```

### 5.3 信号量（控制并发数）

```java
import org.apache.curator.framework.recipes.locks.InterProcessSemaphoreV2;
import org.apache.curator.framework.recipes.locks.Lease;

// 最多允许 3 个客户端同时执行
InterProcessSemaphoreV2 semaphore =
        new InterProcessSemaphoreV2(client, "/semaphores/limited-ops", 3);

Lease lease = semaphore.acquire();  // 获取一个租约
try {
    // 执行业务
} finally {
    semaphore.returnLease(lease);   // 归还租约
}
```

## 六、Leader 选举

```java
import org.apache.curator.framework.recipes.leader.LeaderSelector;
import org.apache.curator.framework.recipes.leader.LeaderSelectorListener;
import org.apache.curator.framework.recipes.leader.LeaderSelectorListenerAdapter;

LeaderSelector selector = new LeaderSelector(client, "/election/master",
        new LeaderSelectorListenerAdapter() {
            @Override
            public void takeLeadership(CuratorFramework client) throws Exception {
                System.out.println("我成为 Leader 了！开始执行 Leader 专属任务...");

                // 这里做 Leader 该做的事，比如分配任务、协调其他节点
                while (true) {
                    Thread.sleep(5000);
                    System.out.println("Leader 心跳...");
                }

                // 当这个方法返回或抛出异常时，会触发重新选举
            }
        });

// 设置为：当前 Leader 退出后自动重新参与选举
selector.autoRequeue();
selector.start();
```

**原理**：Curator 的 Leader 选举背后用的是 EPHEMERAL_SEQUENTIAL 节点。所有候选人创建临时顺序节点，序号最小的当选。

## 七、一个完整的实战例子：简单的配置中心

下面是一个完整可运行的例子，演示了如何用 Curator 实现一个简易配置中心：

```java
import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.framework.recipes.cache.NodeCache;
import org.apache.curator.retry.ExponentialBackoffRetry;

public class SimpleConfigCenter {
    private CuratorFramework client;
    private String dbUrl = "jdbc:mysql://localhost:3306/default";
    private int maxConnections = 10;

    public void init() throws Exception {
        // 1. 连接 ZK
        client = CuratorFrameworkFactory.builder()
                .connectString("127.0.0.1:2181")
                .sessionTimeoutMs(15000)
                .retryPolicy(new ExponentialBackoffRetry(1000, 3))
                .build();
        client.start();

        // 2. 确保配置节点存在（如不存在则创建默认值）
        if (client.checkExists().forPath("/config/app") == null) {
            client.create()
                  .creatingParentsIfNeeded()
                  .forPath("/config/app/db-url", dbUrl.getBytes());
            client.create()
                  .creatingParentsIfNeeded()
                  .forPath("/config/app/max-connections",
                           String.valueOf(maxConnections).getBytes());
        }

        // 3. 加载初始配置
        loadConfig();

        // 4. 监听配置变化
        watchConfigChanges();
    }

    private void loadConfig() throws Exception {
        dbUrl = new String(client.getData().forPath("/config/app/db-url"));
        maxConnections = Integer.parseInt(
                new String(client.getData().forPath("/config/app/max-connections")));
        System.out.println("配置加载完成：");
        System.out.println("  db-url: " + dbUrl);
        System.out.println("  max-connections: " + maxConnections);
    }

    private void watchConfigChanges() throws Exception {
        // 监听 db-url 的变化
        NodeCache dbUrlCache = new NodeCache(client, "/config/app/db-url");
        dbUrlCache.getListenable().addListener(() -> {
            dbUrl = new String(client.getData().forPath("/config/app/db-url"));
            System.out.println("db-url 已更新：" + dbUrl);
        });
        dbUrlCache.start();

        // 监听 max-connections 的变化
        NodeCache maxConnCache = new NodeCache(client, "/config/app/max-connections");
        maxConnCache.getListenable().addListener(() -> {
            maxConnections = Integer.parseInt(
                new String(client.getData().forPath("/config/app/max-connections")));
            System.out.println("max-connections 已更新：" + maxConnections);
        });
        maxConnCache.start();
    }

    public static void main(String[] args) throws Exception {
        SimpleConfigCenter center = new SimpleConfigCenter();
        center.init();

        // 持续运行，等待观察配置变化
        System.out.println("\n配置中心已启动，等待配置变更...\n");
        Thread.sleep(600000);
    }
}
```

**验证步骤：**

1. 启动 ZooKeeper 集群
2. 运行上面的程序
3. 在 zkCli 中修改配置：

```bash
set /config/app/db-url "jdbc:mysql://prod-server:3306/proddb"
```

你会立即看到程序输出 `db-url 已更新：jdbc:mysql://prod-server:3306/proddb`。

## 八、Curator 生产最佳实践

1. **永远用 `close()` 释放资源**：客户端、Cache、Lock、Selector 用完都要关
2. **用 `ExponentialBackoffRetry`**：指数退避比固定间隔重试更健壮
3. **配置 sessionTimeout 至少 15 秒**：给网络抖动留缓冲
4. **不要在生产环境 `deleteAll`**：容易误删不该删的数据
5. **加锁后务必在 finally 里 release**：防止死锁
6. **Curator 回调里不要做耗时操作**：会阻塞 Watcher 线程

## 九、总结

这篇我们学会了：

1. Curator 连接和重试配置
2. Fluent API 的 CRUD 操作
3. NodeCache / PathChildrenCache / TreeCache 三种自动监听
4. 分布式锁（互斥锁、读写锁、信号量）
5. Leader 选举
6. 一个完整的配置中心例子

基本上有了这些能力，你就能应对 90% 的 ZooKeeper 开发需求了。

下一篇，我们站在更高维度，看看 ZooKeeper 在各��分布式场景中到底怎么用——从服务发现到分布式 ID 生成。
