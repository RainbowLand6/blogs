---
date: 2026-07-16
is_published: true
title: Docker Linux 小白教程
tags:
  - Docker
  - Linux
  - 容器
  - 教程
categories:
  - Docker
---

# Docker Linux 小白教程

这是一套以 Linux 服务器为例的 Docker 入门教程。主线使用 Ubuntu/Debian 的命令；CentOS、Rocky Linux、AlmaLinux 等系统的概念和 Docker 命令相同，只是安装软件时使用 `dnf` 或 `yum`。

学完后，你可以在 Linux 上安装 Docker、运行服务、编写镜像、使用 Compose 管理多容器应用，并部署一个 Java 服务。

## 阅读顺序

### 入门

1. [[01-什么是Docker]]
2. [[02-Ubuntu安装Docker]]
3. [[03-镜像与容器基础命令]]
4. [[04-数据卷与持久化]]
5. [[05-端口与容器网络]]

### 构建与部署

6. [[06-编写第一个Dockerfile]]
7. [[07-Docker-Compose管理多容器应用]]
8. [[08-Docker部署Java-Spring-Boot应用]]
9. [[09-日志资源与日常运维]]
10. [[10-Docker常见问题与排障清单]]

## 环境约定

- Linux：Ubuntu 22.04 或 Ubuntu 24.04
- 用户：有 `sudo` 权限的普通用户
- 架构：x86_64 或 ARM64 均可
- 命令：默认在 Linux Shell 中执行

> 学习阶段可以在云服务器或虚拟机操作。生产环境运行容器前，请先确认防火墙、磁盘、备份、镜像来源和最小权限策略。
