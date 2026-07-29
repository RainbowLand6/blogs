---
date: 2026-07-14
is_published: true
title: 09-构建 Java 项目并发布制品
tags:
  - Jenkins
  - Java
  - Maven
  - 制品
categories:
  - Jenkins 教程
---

# 09-构建 Java 项目并发布制品

本章以 Maven 项目为例，完成“拉代码 -> 测试 -> 打包 -> 保存 jar”的持续集成流程。

## 推荐使用 Maven Wrapper

项目中若有 `mvnw` 和 `.mvn/`，优先使用：

```bash
./mvnw test
./mvnw package
```

这样项目会固定 Maven 版本，减少“本地能跑、Jenkins 不能跑”的环境差异。

## Jenkinsfile 示例

```groovy
pipeline {
    agent {
        docker {
            image 'maven:3.9-eclipse-temurin-21'
            args '-v $HOME/.m2:/root/.m2'
        }
    }

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    stages {
        stage('检出') {
            steps {
                checkout scm
            }
        }

        stage('测试') {
            steps {
                sh 'mvn -B test'
            }
        }

        stage('打包') {
            steps {
                sh 'mvn -B package -DskipTests'
            }
        }
    }

    post {
        always {
            junit allowEmptyResults: true, testResults: 'target/surefire-reports/*.xml'
            archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
        }
    }
}
```

## 每一步产出什么

| 阶段 | 主要动作 | 成功标志 |
| --- | --- | --- |
| 检出 | 获取当前提交的代码。 | Workspace 中出现项目文件。 |
| 测试 | 执行单元测试。 | Jenkins 测试趋势中有结果。 |
| 打包 | 生成 jar。 | `target/` 目录出现 jar。 |
| post | 保存测试报告和 jar。 | 构建详情可查看报告和 Artifacts。 |

## 两个容易忽略的点

### Docker Agent 不是自动可用的

这个示例要求**执行该流水线的节点能运行 Docker**，并安装 Docker Pipeline 插件。若 Jenkins 本身运行在 Docker 容器内，还需要额外设计 Docker 访问方式；初学时可以先用已安装 JDK、Maven 的独立 Agent。

### 制品不是永久存档

Jenkins 可能会按保留策略清理旧构建。关键版本的 jar 应推送到 Maven 私服或制品仓库，不要长期只存放在 Jenkins。

## 本章完成标志

一次构建后，Jenkins 中能看到测试结果，并可以从 Artifacts 下载本次生成的 jar 文件。
