---
date: 2026-07-14
is_published: true
title: 05-理解 Pipeline 与 Jenkinsfile
tags:
  - Jenkins
  - Pipeline
  - Jenkinsfile
categories:
  - Jenkins 教程
---

# 05-理解 Pipeline 与 Jenkinsfile

## 为什么不一直用自由风格任务

自由风格任务的配置主要保存在 Jenkins 页面中。它的问题是：

- 配置不容易和代码一起审查。
- 换 Jenkins 或恢复环境时迁移麻烦。
- 多个任务的步骤容易复制粘贴后逐渐不一致。

Pipeline 把流程写进代码，通常放在项目根目录的 `Jenkinsfile` 中并提交到 Git。

```text
代码变更 + Jenkinsfile 变更 = 同一次代码评审
```

## Declarative 和 Scripted 两种语法

| 类型 | 特点 | 建议 |
| --- | --- | --- |
| Declarative Pipeline | 结构固定、可读性好、校验更友好。 | 初学者和大多数团队优先使用。 |
| Scripted Pipeline | 更接近 Groovy 编程，灵活度高。 | 有复杂动态逻辑时再使用。 |

本教程使用 Declarative Pipeline。

## 最小 Jenkinsfile

```groovy
pipeline {
    agent any

    stages {
        stage('Hello') {
            steps {
                sh 'echo Hello, Jenkins!'
            }
        }
    }
}
```

关键字解释：

| 写法 | 含义 |
| --- | --- |
| `pipeline` | 整条流水线的外层定义。 |
| `agent any` | 找任意可用执行节点运行。 |
| `stages` | 阶段集合。 |
| `stage` | 一个可在界面中看到的阶段，例如“测试”。 |
| `steps` | 阶段内真正执行的动作。 |
| `sh` | 在 Linux Shell 中执行命令。 |

## 一个常用骨架

```groovy
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('Test') {
            steps {
                sh './mvnw test'
            }
        }
        stage('Package') {
            steps {
                sh './mvnw package -DskipTests'
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
        }
    }
}
```

`post { always { ... } }` 表示无论成功还是失败都执行，适合收集日志、测试报告和构建产物。

## 本章完成标志

你知道 `Jenkinsfile` 应与项目代码一起提交，并能区分 `stage`（阶段）与 `steps`（具体动作）。
