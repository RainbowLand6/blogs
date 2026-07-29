---
date: 2026-07-16
is_published: true
title: Docker Compose 管理多容器应用
tags: [Docker, Linux, Compose]
categories: [Docker]
---

# Docker Compose 管理多容器应用

目标：用一个 `compose.yaml` 同时启动应用和 MySQL，而不是手工记住多条 `docker run` 命令。

## 1. 为什么使用 Compose

一个 Java 项目常常依赖 MySQL、Redis 等服务。Compose 把镜像、端口、网络、数据卷和环境变量写成可版本管理的配置。

```text
compose.yaml
├── app
└── mysql
```

## 2. 一个最小示例

```yaml
services:
  mysql:
    image: mysql:8.4
    environment:
      MYSQL_DATABASE: demo
      MYSQL_USER: demo
      MYSQL_PASSWORD: change-me
      MYSQL_ROOT_PASSWORD: change-root-password
    volumes:
      - mysql-data:/var/lib/mysql
    networks:
      - app-net

  app:
    image: my-company/demo-app:1.0
    depends_on:
      - mysql
    environment:
      SPRING_DATASOURCE_URL: jdbc:mysql://mysql:3306/demo
      SPRING_DATASOURCE_USERNAME: demo
      SPRING_DATASOURCE_PASSWORD: change-me
    ports:
      - "8080:8080"
    networks:
      - app-net

volumes:
  mysql-data:

networks:
  app-net:
```

`mysql` 是服务名，也是同一网络中应用访问数据库的主机名。

## 3. 常用命令

```bash
docker compose up -d
docker compose ps
docker compose logs -f app
docker compose down
```

更新镜像后：

```bash
docker compose pull
docker compose up -d
```

## 4. `depends_on` 的边界

`depends_on` 只保证容器启动顺序，不保证 MySQL 已经可以接受连接。Java 应用仍要有数据库连接重试，或为数据库配置健康检查后再让应用依赖健康状态。

## 5. 不要把密码提交到 Git

示例中的密码只是占位符。真实项目应使用：

- 部署平台环境变量。
- 单独管理、权限受控的 `.env` 文件。
- Docker secrets、Kubernetes Secret 或云密钥管理服务。

下一篇将使用 Docker 部署一个 Spring Boot JAR。
