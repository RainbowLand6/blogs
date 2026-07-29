---
date: 2026-07-14
is_published: true
title: 07-连接 Git 仓库与自动触发构建
tags:
  - Jenkins
  - Git
  - Webhook
categories:
  - Jenkins 教程
---

# 07-连接 Git 仓库与自动触发构建

从这一章开始，让 Jenkins 从 Git 仓库读取 `Jenkinsfile`。这样构建流程和项目代码会始终保持同步。

## 准备仓库

在项目根目录创建 `Jenkinsfile`：

```groovy
pipeline {
    agent any

    stages {
        stage('检查代码') {
            steps {
                sh 'git rev-parse --short HEAD'
                sh 'echo "正在构建分支：${BRANCH_NAME:-main}"'
            }
        }
    }
}
```

提交并推送：

```bash
git add Jenkinsfile
git commit -m "ci: add Jenkins pipeline"
git push
```

## 创建“流水线脚本来自 SCM”任务

1. 新建一个 **Pipeline** 任务，例如 `demo-service`。
2. 在 Pipeline 区域将 Definition 选为 **Pipeline script from SCM**。
3. SCM 选择 **Git**。
4. 填写仓库地址。
5. 指定分支，例如 `*/main`。
6. Script Path 保持为 `Jenkinsfile`。
7. 保存并点击 **Build Now**。

公开仓库通常不需要凭据；私有仓库需要在 Credentials 中配置 Token 或 SSH 私钥，下一章会介绍。

## 触发方式怎么选

| 方式 | 特点 | 建议 |
| --- | --- | --- |
| 手动 Build Now | 最简单，但依赖人工。 | 学习和临时任务。 |
| Poll SCM | Jenkins 定时检查仓库是否有新提交。 | 没有 Webhook 权限时的备用方案。 |
| Webhook | Git 平台在 push 时通知 Jenkins。 | 团队项目优先使用。 |

不要把 Poll SCM 当作定时构建。它的含义是“定时检查代码是否变更”；要定时执行任务，应使用 Build periodically。

## 配置 Webhook 的思路

不同平台页面名称略有不同，但步骤相同：

1. Jenkins 任务勾选对应的 GitHub/GitLab 触发器。
2. 在 Git 平台仓库的 Webhook 设置中新增回调地址。
3. 选择 Push events。
4. 推送一次代码，在 Jenkins 的构建历史中确认自动出现新构建。

如果 Jenkins 在内网，Git 平台无法访问它，Webhook 就不会生效。此时要么给 Jenkins 配置可访问地址，要么暂时使用 Poll SCM。

## 本章完成标志

Pipeline 任务能从 Git 拉取项目中的 `Jenkinsfile` 并成功运行。若已配置 Webhook，推送一次新提交后会自动触发构建。
