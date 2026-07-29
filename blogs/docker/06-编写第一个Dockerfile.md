---
date: 2026-07-16
is_published: true
title: 编写第一个 Dockerfile
tags: [Docker, Linux, Dockerfile]
categories: [Docker]
---

# 编写第一个 Dockerfile

目标：把一个简单 Web 页面打包为自己的 Nginx 镜像，理解 Dockerfile 的基本指令。

## 1. 创建项目文件

```text
dockerfile-demo/
├── Dockerfile
└── index.html
```

`index.html`：

```html
<h1>Hello from my Docker image</h1>
```

`Dockerfile`：

```dockerfile
FROM nginx:alpine

COPY index.html /usr/share/nginx/html/index.html

EXPOSE 80
```

## 2. 三个指令

| 指令 | 作用 |
|---|---|
| `FROM` | 指定基础镜像 |
| `COPY` | 把构建上下文中的文件复制到镜像 |
| `EXPOSE` | 声明容器预期使用的端口，不会自动发布端口 |

## 3. 构建与运行

在 `dockerfile-demo` 目录执行：

```bash
docker build -t my-nginx:1.0 .
docker image ls my-nginx

docker run -d --rm \
  --name my-nginx-demo \
  -p 8080:80 \
  my-nginx:1.0
```

访问：

```bash
curl http://127.0.0.1:8080
```

## 4. `.dockerignore`

构建时 Docker 会把当前目录作为构建上下文发送给 Docker 引擎。不要把无关或敏感文件带进去。

新建 `.dockerignore`：

```text
.git
.idea
target
node_modules
.env
*.log
```

## 5. Dockerfile 的基础原则

- 使用明确、可信的基础镜像标签。
- 只复制运行需要的文件。
- 不把密码、私钥、`.env` 写进镜像。
- 能使用非 root 用户时就不要以 root 运行。
- 镜像构建完成后实际运行验证，而不是只看构建成功。

下一篇用 Compose 把多个容器作为一个应用来管理。
