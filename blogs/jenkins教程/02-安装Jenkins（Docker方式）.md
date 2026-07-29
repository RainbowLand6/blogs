---
date: 2026-07-14
is_published: true
title: 02-安装 Jenkins（Docker 方式）
tags:
  - Jenkins
  - Docker
  - 安装
categories:
  - Jenkins 教程
---

# 02-安装 Jenkins（Docker 方式）

本章用 Docker 启动 Jenkins。它比直接安装更容易清理、迁移和升级。

## 开始前检查

需要：

- 已安装 Docker Desktop，或 Linux Docker Engine。
- 本机 `8080` 和 `50000` 端口没有被占用。

验证 Docker：

```bash
docker version
docker run --rm hello-world
```

## 启动 Jenkins

先创建一个持久化数据卷：

```bash
docker volume create jenkins_home
```

再启动容器：

```bash
docker run -d \
  --name jenkins \
  --restart unless-stopped \
  -p 8080:8080 \
  -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  jenkins/jenkins:lts-jdk21
```

参数说明：

| 参数 | 用途 |
| --- | --- |
| `-p 8080:8080` | 让浏览器访问 Jenkins 页面。 |
| `-p 50000:50000` | 供入站 Agent 连接 Controller。当前阶段可以保留。 |
| `-v jenkins_home:/var/jenkins_home` | 保存 Jenkins 配置、插件和构建记录。删除容器后数据仍在。 |
| `--restart unless-stopped` | 机器重启后自动恢复 Jenkins。 |

## 获取初始管理员密码

```bash
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

复制输出内容，浏览器打开 `http://localhost:8080`，粘贴密码并继续。

## 首次安装怎么选

首次进入时建议选择 **Install suggested plugins**。它会安装常用的 Pipeline、Git、Credentials 等插件。

随后创建一个管理员账户。不要长期使用默认 `admin` 用户名，也不要使用弱密码。

## 常用容器命令

```bash
# 查看运行状态
docker ps --filter name=jenkins

# 查看实时日志
docker logs -f jenkins

# 停止和启动
docker stop jenkins
docker start jenkins

# 删除容器；数据卷不会被删除
docker rm -f jenkins
```

## 常见问题

### 页面打不开

先执行：

```bash
docker ps -a --filter name=jenkins
docker logs jenkins
```

若容器没有运行，日志通常会说明原因。最常见的是端口被占用或 Docker 的内存不足。

### 忘记初始密码

初始密码只用于首次解锁。已经创建管理员账户后，应使用账户密码登录；忘记账户密码时，不要随意删除 `jenkins_home` 数据卷。

## 本章完成标志

浏览器能打开 Jenkins，且你已创建自己的管理员账户。
