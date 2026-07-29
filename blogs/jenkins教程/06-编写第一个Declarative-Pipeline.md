---
date: 2026-07-14
is_published: true
title: 06-编写第一个 Declarative Pipeline
tags:
  - Jenkins
  - Pipeline
  - Jenkinsfile
categories:
  - Jenkins 教程
---

# 06-编写第一个 Declarative Pipeline

本章不依赖 Git 仓库，直接在 Jenkins 页面运行第一条 Pipeline。

## 新建 Pipeline 任务

1. 点击 **New Item**。
2. 名称输入 `first-pipeline`。
3. 选择 **Pipeline**。
4. 在 Pipeline 区域将 Definition 选择为 **Pipeline script**。
5. 粘贴下方脚本并保存。

```groovy
pipeline {
    agent any

    stages {
        stage('准备') {
            steps {
                echo '开始执行流水线'
                sh 'java -version || true'
            }
        }

        stage('构建') {
            steps {
                sh 'mkdir -p output && echo "build-${BUILD_NUMBER}" > output/result.txt'
            }
        }

        stage('验证') {
            steps {
                sh 'test -f output/result.txt'
                sh 'cat output/result.txt'
            }
        }
    }

    post {
        success {
            echo '流水线执行成功'
        }
        failure {
            echo '流水线执行失败，请查看失败阶段的控制台日志'
        }
        always {
            archiveArtifacts artifacts: 'output/**', allowEmptyArchive: true
            cleanWs()
        }
    }
}
```

> 如果页面提示找不到 `cleanWs`，说明没有安装 Workspace Cleanup 插件。先删除这一行继续练习，或安装该插件后重试。

## 运行并观察

点击 **Build Now**。任务页面会显示“准备、构建、验证”三个阶段。

构建成功后，在构建详情的 **Artifacts** 中应能下载 `result.txt`。

## 常见错误

### `sh: not found`

当前执行节点可能是 Windows。解决办法：

- 改用 Linux Agent 或 Jenkins Linux 容器。
- 或将 `sh '命令'` 改为 `bat '命令'`，并使用 Windows 命令语法。

### `java: not found`

示例中的 `|| true` 会忽略此错误，所以流水线仍能继续。真正构建 Java 项目前，需要在 Agent 上安装 Java，或使用包含 JDK 的 Docker Agent。

## 本章完成标志

`first-pipeline` 成功运行，页面中能看到三个阶段，且构建详情包含 `result.txt` 制品。
