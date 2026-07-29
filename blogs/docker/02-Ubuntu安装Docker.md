---
date: 2026-07-16
is_published: true
title: Ubuntu 安装 Docker
tags: [Docker, Linux, Ubuntu, 安装]
categories: [Docker]
---

# Ubuntu 安装 Docker

目标：在 Ubuntu 上通过 Docker 官方 apt 仓库安装 Docker Engine 和 Compose 插件，并运行第一个容器。

## 1. 确认系统和架构

```bash
cat /etc/os-release
uname -m
```

本篇适用于 Ubuntu 22.04、24.04 等较新的 LTS 版本。使用 CentOS、Rocky Linux、AlmaLinux 时，请按对应系统的官方安装文档添加 Docker 仓库；后续 `docker` 命令相同。

## 2. 卸载冲突的软件包

新机器通常没有这些包；已有旧 Docker 时，先执行：

```bash
sudo apt-get remove -y docker.io docker-compose docker-compose-v2 docker-doc podman-docker containerd runc
```

这一步不会删除 `/var/lib/docker` 中已有的镜像、容器和卷。生产服务器执行前，应先确认是否有正在运行的业务容器。

## 3. 添加 Docker 官方 apt 仓库

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

## 4. 安装 Docker

```bash
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

验证：

```bash
sudo docker version
sudo docker compose version
sudo systemctl status docker --no-pager
```

## 5. 运行第一个容器

```bash
sudo docker run --rm hello-world
```

看到欢迎信息表示 Docker 已能拉取镜像并运行容器。

## 6. 让普通用户使用 Docker

不想每次都写 `sudo`，可以把当前用户加入 `docker` 组：

```bash
sudo usermod -aG docker $USER
newgrp docker
docker ps
```

`docker` 组拥有接近 root 的主机控制能力。只应把可信运维用户加入该组；多人服务器不要为了省事把所有账号加入。

下一篇学习镜像和容器最常用的命令。
