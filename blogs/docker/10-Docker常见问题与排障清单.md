---
date: 2026-07-16
is_published: true
title: Docker 常见问题与排障清单
tags: [Docker, Linux, 排障]
categories: [Docker]
---

# Docker 常见问题与排障清单

目标：容器启动失败、端口访问不了或磁盘满时，按顺序缩小问题范围。

## 1. 先看容器状态

```bash
docker ps -a
docker logs --tail 200 <容器名>
```

| 状态 | 常见含义 |
|---|---|
| `Up` | 正在运行 |
| `Exited` | 进程已退出，先看日志 |
| `Restarting` | 不断崩溃重启，先看第一次异常 |
| `unhealthy` | 健康检查失败，应用进程不一定已退出 |

## 2. 常见现象速查

| 现象 | 常见原因 | 第一动作 |
|---|---|---|
| `permission denied` 访问 Docker | 用户不在 `docker` 组 | 使用 `sudo` 或重新登录后检查用户组 |
| `port is already allocated` | 主机端口已被占用 | `ss -lntp` 查占用端口 |
| 容器刚启动就退出 | 启动命令错误、环境变量缺失 | 看 `docker logs` |
| 服务外部访问不了 | 未映射端口、防火墙或安全组未放行 | 先在服务器本机 `curl` |
| 容器内连不上 MySQL | 用了 `localhost`、网络不同 | 检查 Compose 服务名和网络 |
| 数据重建后丢失 | 没有挂载 Volume | 检查 `docker inspect` 的 Mounts |
| 磁盘空间不足 | 镜像、日志、卷长期累积 | `docker system df` |

## 3. 网络排查顺序

1. 服务容器是否 `Up`。
2. 应用是否在容器内监听正确端口。
3. Docker 是否正确发布 `-p 主机端口:容器端口`。
4. 在 Linux 主机执行 `curl 127.0.0.1:主机端口`。
5. 检查 `ufw`、firewalld 和云安全组。
6. 最后才检查域名、Nginx 和公网链路。

## 4. 四条生产底线

- 不使用 `latest` 部署关键服务。
- 不将数据库端口直接映射到公网。
- 数据库数据使用 Volume，并有独立备份与恢复演练。
- 镜像、Compose 文件、环境变量和发布操作都应可追踪、可回滚。

## 5. 学习完成检查

完成以下事情，说明你已经掌握 Docker Linux 入门主线：

- Ubuntu 上安装 Docker 和 Compose。
- 用镜像启动 Nginx，并通过端口访问。
- 用 Volume 持久化一个数据库容器的数据。
- 编写 Dockerfile 构建镜像。
- 用 Compose 同时启动 Java 应用和 MySQL。

下一步可继续学习镜像仓库、Nginx 反向代理、HTTPS、CI/CD、Docker Swarm 或 Kubernetes。先把镜像、容器、网络、卷和 Compose 用稳，比过早堆叠编排工具更重要。
