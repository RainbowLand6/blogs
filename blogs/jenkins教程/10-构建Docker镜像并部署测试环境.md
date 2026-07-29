---
date: 2026-07-14
is_published: true
title: 10-构建 Docker 镜像并部署测试环境
tags:
  - Jenkins
  - Docker
  - 部署
categories:
  - Jenkins 教程
---

# 10-构建 Docker 镜像并部署测试环境

这一章演示一个可理解的部署流程：测试通过后构建镜像，推送到镜像仓库，再通过 SSH 让测试服务器更新容器。

## Dockerfile 示例

放在 Java 项目根目录：

```dockerfile
FROM eclipse-temurin:21-jre

WORKDIR /app
COPY target/*.jar app.jar

EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

## 部署前准备

需要提前完成：

- Jenkins 执行节点能使用 Docker CLI。
- Jenkins 中已保存镜像仓库账号密码，ID 例如 `docker-registry-test`。
- Jenkins 中已保存测试服务器 SSH 私钥，ID 例如 `test-server-ssh`。
- 测试服务器已安装 Docker，且部署用户有运行 Docker 的权限。

## Jenkinsfile 核心示例

以下示例假设镜像仓库地址为 `registry.example.com`，按你的实际地址替换：

```groovy
pipeline {
    agent any

    environment {
        IMAGE_NAME = 'registry.example.com/demo/demo-service'
        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {
        stage('测试与打包') {
            steps {
                sh 'mvn -B package'
            }
        }

        stage('构建镜像') {
            steps {
                sh 'docker build -t $IMAGE_NAME:$IMAGE_TAG .'
            }
        }

        stage('推送镜像') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker-registry-test',
                    usernameVariable: 'REGISTRY_USER',
                    passwordVariable: 'REGISTRY_PASSWORD'
                )]) {
                    sh '''
                        echo "$REGISTRY_PASSWORD" | docker login registry.example.com \
                          -u "$REGISTRY_USER" --password-stdin
                        docker push "$IMAGE_NAME:$IMAGE_TAG"
                    '''
                }
            }
        }

        stage('部署测试环境') {
            steps {
                sshagent(credentials: ['test-server-ssh']) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no deploy@test.example.com "
                          docker pull $IMAGE_NAME:$IMAGE_TAG &&
                          docker rm -f demo-service || true &&
                          docker run -d --name demo-service -p 8080:8080 $IMAGE_NAME:$IMAGE_TAG
                        "
                    '''
                }
            }
        }
    }
}
```

## 生产环境不能直接照抄

上面的方式适合学习和简单测试环境。生产发布至少应补充：

- 使用不可变标签，例如 Git commit SHA，而不是只用 `latest`。
- 部署前人工确认。
- 健康检查和失败回滚。
- 主机密钥校验，不能长期使用 `StrictHostKeyChecking=no`。
- 镜像扫描、权限隔离和审计。

## 本章完成标志

测试环境能拉取本次构建的指定镜像标签，并启动新版本容器；你能在 Jenkins 日志中追踪本次使用的镜像版本。
