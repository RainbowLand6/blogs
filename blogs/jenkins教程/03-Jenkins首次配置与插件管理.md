---
date: 2026-07-14
is_published: true
title: 03-Jenkins 首次配置与插件管理
tags:
  - Jenkins
  - 插件
  - 配置
categories:
  - Jenkins 教程
---

# 03-Jenkins 首次配置与插件管理

## 认识首页

登录后，最常用的入口有：

| 入口 | 用途 |
| --- | --- |
| New Item | 新建任务。 |
| Build History | 查看最近的构建结果。 |
| Manage Jenkins | 管理插件、用户、节点和系统设置。 |
| Manage Credentials | 管理 Git Token、SSH 私钥、镜像仓库密码等敏感信息。 |

## 安装必需插件

进入 **Manage Jenkins -> Plugins**，在 Available plugins 中搜索并安装：

| 插件 | 用途 |
| --- | --- |
| Pipeline | 编写和运行流水线。通常已随推荐插件安装。 |
| Git | 拉取 Git 仓库。 |
| GitHub Branch Source | 使用 GitHub 多分支流水线时需要。 |
| Credentials Binding | 将凭据安全地注入构建环境。 |
| Docker Pipeline | 在流水线中使用 Docker。 |
| Workspace Cleanup | 构建后清理工作目录。 |

原则：**只装当前需要的插件。** 插件越多，升级和排错成本越高。

## 配置全局工具

进入 **Manage Jenkins -> Tools**，可配置 JDK、Git、Maven、Node.js 等工具。

初学阶段有两种做法：

1. 工具已安装在 Jenkins 容器或 Agent 中：填写安装目录。
2. 使用 Docker Agent：每次构建使用带好工具的临时容器。

第二种更容易保证环境一致，后续第 6 章会演示。

## 系统配置中先关注什么

进入 **Manage Jenkins -> System**，初学时重点看：

- Jenkins URL：设置为团队能访问的地址。发通知、生成回调链接时会用到。
- Number of executors：Controller 不建议承担大量构建。学习环境可以保持默认，生产环境通常让 Agent 执行构建。
- GitHub / GitLab 配置：接入 Webhook 后再配置。

## 插件升级建议

- 有可用备份后再升级。
- 不要在发布高峰期一次升级大量插件。
- 升级后至少跑一条代表性流水线。
- 看到依赖升级提示时，优先按 Jenkins 页面给出的依赖关系处理。

## 本章完成标志

你能找到插件管理、全局工具和凭据管理入口，并确认 Git 与 Pipeline 插件已可用。
