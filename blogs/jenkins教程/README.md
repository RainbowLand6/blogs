---
date: 2026-07-14
is_published: true
title: Jenkins 从入门到进阶教程
tags:
  - Jenkins
  - CI/CD
  - 教程
categories:
  - Jenkins 教程
---

# Jenkins 从入门到进阶教程

这是一套给初学者的 Jenkins 实战教程。目标不是记住一堆按钮，而是能够自己搭建一条可维护的 CI/CD 流水线。

## 学完后你能做到什么

- 用 Docker 在本地启动 Jenkins。
- 创建任务，在代码提交后自动构建和测试。
- 编写并理解 Declarative Pipeline 风格的 `Jenkinsfile`。
- 使用凭据安全地访问 Git 仓库、镜像仓库和服务器。
- 构建 Docker 镜像，并部署到测试环境。
- 排查常见失败，完成 Jenkins 的备份、恢复和日常维护。

## 阅读顺序

### 入门

1. [[01-什么是Jenkins]]
2. [[02-安装Jenkins（Docker方式）]]
3. [[03-Jenkins首次配置与插件管理]]
4. [[04-创建第一个自由风格任务]]

### 核心

5. [[05-理解Pipeline与Jenkinsfile]]
6. [[06-编写第一个Declarative-Pipeline]]
7. [[07-连接Git仓库与自动触发构建]]
8. [[08-管理凭据与环境变量]]

### 实战与进阶

9. [[09-构建Java项目并发布制品]]
10. [[10-构建Docker镜像并部署测试环境]]
11. [[11-多分支流水线与共享库]]
12. [[12-安全、备份、监控与故障排查]]

## 练习项目约定

后续示例以 Maven Java 项目为例，目录大致如下：

```text
demo-service/
├── src/
├── pom.xml
├── Dockerfile
└── Jenkinsfile
```

不会 Java 也没关系。重点是理解 Jenkins 在每一步执行了什么命令、输入是什么、成功后产出了什么。

> 学习建议：先在本机或测试服务器练习。不要一开始把 Jenkins 接到生产环境，更不要把生产密码直接写进 `Jenkinsfile`。
