---
date: 2026-06-09
is_published: true
title: Redis集群JedisNoReachableClusterNodeException问题排查与解决
tags:
  - Redis
  - Jedis
  - 集群
categories:
  - Redis
---

## 一、问题现象

在生产环境中，Java应用使用Jedis连接Redis集群时出现以下异常：

```text
    redis.clients.jedis.exceptions.JedisNoReachableClusterNodeException: No reachable node in cluster

```

应用无法连接到Redis集群，导致会话管理、缓存等功能失效。

## 二、环境信息

  * **Redis集群** ：6个节点（3主3从），部署在同一台服务器上

  * **Jedis版本** ：2.x（不支持`--cluster`命令）

  * **Java框架** ：Spring + Shiro

  * **部署方式** ：Redis和应用在同一台服务器


## 三、问题排查过程

### 3.1 检查Redis进程状态

```bash
    ps aux | grep redis-server

```

输出显示所有6个Redis进程都在运行，说明进程正常。

### 3.2 检查集群节点状态

```bash
    redis-cli -h 172.17.140.201 -p 7000 cluster nodes

```

发现部分节点（如7000、7001、7002）显示的是`127.0.0.1`，而其他节点显示的是真实IP`172.17.140.201`，存在**IP地址不一致** 的问题。

### 3.3 检查Java应用配置

查看Spring配置文件：

```xml
    <bean id="jedisCluster" class="redis.clients.jedis.JedisCluster">
        <constructor-arg index="0" type="java.util.Set">
            <set>
                <bean class="redis.clients.jedis.HostAndPort">
                    <constructor-arg name="host" value="${redis.cluster.0.host}"/>
                    <constructor-arg name="port" value="${redis.cluster.0.port}"/>
                </bean>
                <!-- 其他节点配置... -->
            </set>
        </constructor-arg>
    </bean>

```

对应的properties配置：

```properties
    redis.cluster.0.host=127.0.0.1
    redis.cluster.0.port=7000
    # 所有节点都配置为127.0.0.1

```

### 3.4 网络连通性测试

```bash
    telnet 172.17.140.201 7000

```

结果：连接成功，说明网络是通的。

### 3.5 根本原因分析

问题出在**IP地址不匹配** ：

  1. **Redis集群内部** ：部分节点的`bind`配置为`172.17.140.201`，导致集群内部记录的IP是真实IP

  2. **Java应用配置** ：Jedis配置使用的是`127.0.0.1`

  3. **结果** ：Jedis客户端尝试连接`127.0.0.1`，但Redis节点宣告的是`172.17.140.201`，导致连接失败


## 四、解决方案

### 4.1 方案选择

有两种解决方案：

  * **方案A** ：修改Redis集群配置，统一使用`127.0.0.1`（适合应用和Redis在同一台服务器）

  * **方案B** ：修改Java应用配置，统一使用真实IP`172.17.140.201`


本文采用**方案A** ，因为应用和Redis在同一台服务器上。

### 4.2 具体实施步骤

#### 步骤1：修改Redis配置文件

将所有节点的`bind`配置统一改为`127.0.0.1`：

```bash
    cd ~/redis_cluster
    
    # 修改每个节点的bind配置
    for port in 7000 7001 7002 7003 7004 7005; do
      sed -i '/^bind/d' $port/redis.conf
      echo "bind 127.0.0.1" >> $port/redis.conf
    done
    
    # 验证修改结果
    grep "^bind" */redis.conf

```

#### 步骤2：停止所有Redis节点

```bash
    killall redis-server

```

#### 步骤3：清理旧数据

```bash
    cd ~/redis_cluster
    rm -f nodes-*.conf *.rdb

```

#### 步骤4：启动所有节点

```bash
    for port in 7000 7001 7002 7003 7004 7005; do
      redis-server $port/redis.conf
    done

```

#### 步骤5：重新创建集群

由于Redis版本较旧（不支持`--cluster`命令），使用`redis-trib.rb`创建：

```bash
    redis-trib.rb create --replicas 1 \
      127.0.0.1:7000 \
      127.0.0.1:7001 \
      127.0.0.1:7002 \
      127.0.0.1:7003 \
      127.0.0.1:7004 \
      127.0.0.1:7005

```

输入`yes`确认配置。

#### 步骤6：验证集群状态

```bash
    # 查看集群节点
    redis-cli -h 127.0.0.1 -p 7000 cluster nodes
    
    # 测试读写
    redis-cli -h 127.0.0.1 -p 7000 set test "hello"
    redis-cli -h 127.0.0.1 -p 7000 get test

```

#### 步骤7：重启Java应用

```bash
    # Tomcat
    ./shutdown.sh && ./startup.sh
    
    # Spring Boot
    java -jar your-app.jar

```

## 五、注意事项

### 5.1 关于CLUSTER MEET命令

在排查过程中，曾使用`CLUSTER MEET`命令手动将节点加入集群：

```bash
    redis-cli -h 172.17.140.201 -p 7000 CLUSTER MEET 172.17.140.201 7000

```

这个命令可以临时解决问题，但重启后配置会丢失，还是需要修改配置文件。

### 5.2 关于cluster-announce-ip指令

在旧版Redis（<4.0）中，不支持`cluster-announce-ip`指令，只能通过`bind`配置来设置IP地址。

```bash
    # 旧版Redis不支持
    cluster-announce-ip 172.17.140.201  # 会报错 Bad directive

```

### 5.3 关于MOVED错误

测试时可能会出现`(error) MOVED 6918 127.0.0.1:7001`，这是**正常现象** ，表示key被路由到其他节点。JedisCluster会自动处理重定向，应用代码无需关心。

## 六、经验总结

### 6.1 核心要点

  1. **IP地址一致性** ：Redis集群内部宣告的IP必须与客户端连接的IP保持一致

  2. **版本兼容性** ：不同版本的Redis配置指令不同，需要注意版本差异

  3. **配置即代码** ：集群配置应纳入版本管理，避免手动修改


### 6.2 最佳实践

**场景**| **推荐配置**| **原因**  
---|---|---  
应用和Redis同机| Redis监听`127.0.0.1`| 性能最好，无网络开销  
应用和Redis异机| Redis监听`0.0.0.0`，配置真实IP| 跨机通信必须使用真实IP  
生产环境| 使用配置文件统一管理| 可复现、可追溯  
Redis版本| 升级到5.0+| 支持`--cluster`命令和更多特性  
  
### 6.3 快速排查清单

遇到`JedisNoReachableClusterNodeException`时，按以下顺序检查：

  1. ✅ Redis进程是否运行？

  2. ✅ 网络是否通畅？（telnet测试）

  3. ✅ Redis的`bind`配置是否正确？

  4. ✅ Jedis配置的IP是否与Redis宣告的IP一致？

  5. ✅ 防火墙是否开放了端口（包括集群总线端口+10000）？

  6. ✅ Jedis版本是否支持集群模式？


## 七、结语

本次问题的根本原因是Redis集群内部IP地址与客户端配置不一致。通过统一使用`127.0.0.1`，问题得到完全解决。

**一句话总结** ：确保Redis集群所有节点的`bind`配置与Jedis客户端连接的IP地址保持一致，问题即可解决。