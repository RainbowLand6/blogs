---
date: 2026-07-16
is_published: true
title: Docker 部署 Java Spring Boot 应用
tags: [Docker, Linux, Java, Spring Boot]
categories: [Docker]
---

# Docker 部署 Java Spring Boot 应用

目标：将一个已经打包好的 Spring Boot JAR 运行在 Docker 容器中。

## 1. 先在本机打包

项目根目录执行：

```bash
mvn clean package -DskipTests
```

假设产物是：

```text
target/demo-app-1.0.0.jar
```

## 2. 编写 Dockerfile

```dockerfile
FROM eclipse-temurin:21-jre

WORKDIR /app

COPY target/demo-app-1.0.0.jar app.jar

RUN useradd --system --uid 10001 appuser
USER appuser

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

这里使用 JRE 镜像而不是 JDK 镜像，体积更小；同时用非 root 用户运行应用。

## 3. 构建和启动

```bash
docker build -t demo-app:1.0.0 .

docker run -d \
  --name demo-app \
  --restart unless-stopped \
  -p 8080:8080 \
  -e SPRING_PROFILES_ACTIVE=prod \
  -e JAVA_TOOL_OPTIONS="-XX:MaxRAMPercentage=75.0" \
  demo-app:1.0.0
```

查看启动日志：

```bash
docker logs -f --tail 100 demo-app
```

## 4. 多阶段构建

如果 Linux 服务器没有 Maven，也可以让 Docker 在构建时完成打包：

```dockerfile
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /build
COPY pom.xml .
RUN mvn -q -DskipTests dependency:go-offline
COPY src ./src
RUN mvn -q -DskipTests package

FROM eclipse-temurin:21-jre
WORKDIR /app
COPY --from=build /build/target/*.jar app.jar
RUN useradd --system --uid 10001 appuser
USER appuser
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

## 5. 部署更新

不要在运行中的容器里覆盖 JAR。正确流程是：

```text
构建新镜像 -> 启动新容器 -> 健康检查 -> 切换流量 -> 停止旧容器
```

单机学习环境可直接停止旧容器再启动新容器；生产环境应结合反向代理、健康检查和回滚方案。

下一篇讲查看日志、限制资源和日常清理。
