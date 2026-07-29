---
date: 2026-07-10
is_published: true
title: Nginx从入门到进阶
tags:
  - Nginx
categories:
  - 服务器
---

# Nginx 从入门到进阶

这篇文章面向小白。

目标很明确：看完后，你至少要知道 Nginx 是什么、能做什么、怎么安装、怎么配置、怎么使用、怎么进阶、常见问题怎么排查。

如果你以前没接触过 Nginx，也可以直接往下看。

---

## 一、先说人话：Nginx 到底是什么

Nginx 是一个高性能的 Web 服务器，也常被拿来做反向代理、负载均衡和静态资源服务。

如果这句话你一眼没看懂，也没关系。先把它理解成一句更接地气的话：

**Nginx 是网站和后端服务前面的“入口管理员”。**

用户访问网站时，请求通常可以先到 Nginx，然后由 Nginx 决定：

- 这个请求是不是直接返回网页
- 这个请求是不是转发给后端程序
- 这个请求是不是图片、JS、CSS 这种静态文件
- 这个请求是不是该走 HTTPS
- 这个请求是不是要分发到多个后端机器

所以，Nginx 很少负责具体业务逻辑，但经常负责“接住流量”和“把流量安排明白”。

---

## 二、Nginx 能做什么

先记住最常见的 6 个用途。

### 1. 部署静态网站

比如前端项目打包后得到：

- `index.html`
- `app.js`
- `style.css`
- 图片、字体文件

这时 Nginx 可以直接把这些文件提供给浏览器访问。

### 2. 反向代理

比如你的 Java、Node.js、Python 服务跑在：

```text
127.0.0.1:8080
```

用户不直接访问它，而是先访问 Nginx，再由 Nginx 转发给后端。

### 3. 负载均衡

后端不止一台机器时，Nginx 可以把请求分发给多台服务。

### 4. HTTPS 入口

很多网站的 SSL 证书配置、HTTP 跳 HTTPS，都是放在 Nginx 上做的。

### 5. 统一域名入口

你可以让多个系统统一从一个入口进来，比如：

- `example.com`
- `api.example.com`
- `admin.example.com`

### 6. 做一些基础安全和限制

比如：

- 限制上传大小
- 限制访问频率
- 屏蔽某些路径
- 隐藏后端真实地址

---

## 三、几个必须先懂的概念

### 1. Web 服务器

简单说，就是“能对浏览器请求做出响应的服务”。

Nginx 本身就是 Web 服务器。

### 2. 静态资源

指的是已经存在的文件，浏览器直接拿去显示就行，比如：

- HTML
- CSS
- JS
- 图片
- 音视频

### 3. 动态服务

指的是后端程序现算现返回的内容，比如：

- Java Spring Boot
- Node.js Express / NestJS
- Python Flask / Django
- PHP

### 4. 正向代理和反向代理

这个很多人一开始会混。

#### 正向代理

代理的是“客户端”。

比如你通过代理软件去访问外网，本质上是代理帮你出去访问。

#### 反向代理

代理的是“服务端”。

用户以为自己访问的是一个地址，实际上后面可能是 Nginx 再转发给别的服务。

Nginx 最常见的就是做反向代理。

---

## 四、Nginx 的典型工作流程

你可以先把一次请求理解成这样：

1. 用户在浏览器输入网址
2. 请求先到 Nginx
3. Nginx 看配置
4. 如果是静态文件，就直接返回
5. 如果是接口请求，就转发给后端
6. 如果后端有多台，还可以由 Nginx 分配到不同机器
7. 最终把结果返回给浏览器

一句话概括：

**Nginx 负责接请求、分请求、转请求。**

---

## 五、为什么很多项目喜欢用 Nginx

因为它有几个很实际的优点：

- 性能好
- 占用资源相对低
- 配置灵活
- 很适合处理静态资源
- 很适合做统一入口
- 生态成熟，资料多

对小团队和大项目都适用。

---

## 六、Nginx 安装

这一部分分 Windows 和 Linux。

---

## 七、Windows 安装 Nginx

如果你只是本地学习，Windows 最方便。

### 1. 下载

官网：

[Nginx 官网](https://nginx.org/)

一般下载 Windows 版本压缩包即可。

### 2. 解压

比如解压到：

```text
D:\server\nginx
```

解压后常见目录结构大概是：

```text
nginx
├─ conf
├─ html
├─ logs
└─ nginx.exe
```

### 3. 启动

进入目录后运行：

```bash
start nginx
```

或者直接双击 `nginx.exe`。

### 4. 验证

浏览器访问：

```text
http://localhost
```

如果看到欢迎页，说明启动成功。

---

## 八、Linux 安装 Nginx

服务器上更常见的是 Linux。

### 1. Ubuntu / Debian

```bash
apt update
apt install -y nginx
```

### 2. CentOS / Rocky / AlmaLinux

```bash
yum install -y nginx
```

### 3. 启动服务

```bash
systemctl start nginx
```

### 4. 设置开机自启

```bash
systemctl enable nginx
```

### 5. 查看状态

```bash
systemctl status nginx
```

---

## 九、安装后先认识目录结构

不同系统安装位置会略有不同，但核心内容差不多。

常见重点如下。

### 1. 配置文件目录

常见是：

```text
/etc/nginx/
```

或者 Windows 下的：

```text
conf/
```

### 2. 主配置文件

最常见的是：

```text
nginx.conf
```

### 3. 默认网页目录

比如：

```text
html/
```

或者：

```text
/usr/share/nginx/html/
```

### 4. 日志目录

常见是：

- `access.log`
- `error.log`

---

## 十、最常用的几个命令

这部分很实用，建议直接记住。

### 1. 启动 Nginx

```bash
nginx
```

### 2. 停止 Nginx

```bash
nginx -s stop
```

### 3. 优雅退出

```bash
nginx -s quit
```

### 4. 重载配置

```bash
nginx -s reload
```

### 5. 检查配置文件语法

```bash
nginx -t
```

### 6. 查看版本

```bash
nginx -v
```

### 7. 查看详细编译信息

```bash
nginx -V
```

### 8. Linux 下用 systemctl 管理

```bash
systemctl start nginx
systemctl stop nginx
systemctl restart nginx
systemctl reload nginx
systemctl status nginx
```

对新手来说，最重要的两个命令是：

```bash
nginx -t
nginx -s reload
```

因为几乎每次改配置都会用到它们。

---

## 十一、第一个最小可用配置

先看一个最简单的配置：

```nginx
worker_processes  1;

events {
    worker_connections  1024;
}

http {
    server {
        listen 80;
        server_name localhost;

        location / {
            root   html;
            index  index.html index.htm;
        }
    }
}
```

你现在不用逐行全懂，先抓住重点。

### 1. `http`

HTTP 相关配置都写在这里。

### 2. `server`

可以理解成一个网站配置块。

一个 `server` 通常对应一个站点，或者一组访问规则。

### 3. `listen 80`

监听 80 端口。

### 4. `server_name localhost`

匹配访问域名。

### 5. `location /`

匹配路径规则。

### 6. `root html`

指定静态文件目录。

### 7. `index index.html index.htm`

指定默认首页文件。

---

## 十二、从 0 部署一个静态网站

这是最适合小白练手的第一步。

### 1. 准备一个页面

在 Nginx 的网页目录里放一个 `index.html`：

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8" />
  <title>我的第一个 Nginx 网站</title>
</head>
<body>
  <h1>Hello Nginx</h1>
  <p>如果你看到这段内容，说明网站已经跑起来了。</p>
</body>
</html>
```

### 2. 检查配置

```bash
nginx -t
```

### 3. 重载配置

```bash
nginx -s reload
```

### 4. 浏览器访问

```text
http://localhost
```

如果能看到页面，说明你已经跑通了最基础的 Nginx。

---

## 十三、什么是反向代理

这是 Nginx 最核心的用法之一。

假设你的后端程序跑在：

```text
127.0.0.1:8080
```

你不想让用户直接访问 `8080`，而是让用户访问：

```text
http://localhost/api/
```

然后由 Nginx 转过去。

这就是反向代理。

---

## 十四、最简单的反向代理配置

```nginx
server {
    listen 80;
    server_name localhost;

    location /api/ {
        proxy_pass http://127.0.0.1:8080;
    }
}
```

它的含义是：

- 用户访问 `/api/xxx`
- 请求先到 Nginx
- Nginx 再转发给 `127.0.0.1:8080`

你可以先把它理解成：

**Nginx 站前台，后端站后台。**

---

## 十五、前后端分离项目的常见写法

很多项目都是：

- `/` 返回前端页面
- `/api/` 转发给后端接口

示例：

```nginx
server {
    listen 80;
    server_name localhost;

    location /api/ {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location / {
        root   html;
        index  index.html index.htm;
        try_files $uri $uri/ /index.html;
    }
}
```

这里最关键的是两部分。

### 1. `location /api/`

接口请求走后端。

### 2. `location /`

前端页面直接从本地目录返回。

### 3. `try_files $uri $uri/ /index.html`

这是单页应用非常常见的一行。

它的作用是：

1. 先找当前路径对应的文件
2. 找不到再试目录
3. 还找不到就返回 `index.html`

这样可以解决 Vue、React 刷新页面时出现 404 的问题。

---

## 十六、常见配置项，必须知道的有哪些

这一节不求一次背完，但至少要知道它们大概管什么。

### 1. `listen`

监听端口。

```nginx
listen 80;
listen 443 ssl;
```

### 2. `server_name`

匹配域名。

```nginx
server_name example.com www.example.com;
```

### 3. `location`

匹配路径规则。

```nginx
location /api/ { }
location /images/ { }
```

### 4. `root`

指定静态资源根目录。

```nginx
root /data/www/demo;
```

### 5. `index`

默认首页文件。

```nginx
index index.html index.htm;
```

### 6. `proxy_pass`

转发给后端服务。

```nginx
proxy_pass http://127.0.0.1:8080;
```

### 7. `try_files`

按顺序尝试找文件，找不到时走兜底。

```nginx
try_files $uri $uri/ /index.html;
```

### 8. `client_max_body_size`

限制请求体大小，常用于控制上传大小。

```nginx
client_max_body_size 20m;
```

### 9. `error_page`

自定义错误页。

```nginx
error_page 404 /404.html;
error_page 500 502 503 504 /50x.html;
```

### 10. `access_log` 和 `error_log`

访问日志和错误日志。

---

## 十七、`location` 怎么理解

这是很多人学 Nginx 时最容易迷糊的点。

先用简单版本理解。

### 1. 精确匹配

```nginx
location = /login {
    ...
}
```

只有请求路径完全等于 `/login` 才会命中。

### 2. 普通前缀匹配

```nginx
location /api/ {
    ...
}
```

只要路径以 `/api/` 开头，就可能匹配到。

### 3. `^~` 前缀匹配

```nginx
location ^~ /static/ {
    ...
}
```

表示前缀匹配成功后，不再继续走正则匹配。

### 4. 正则匹配

```nginx
location ~ \.php$ {
    ...
}
```

通常用于更灵活的路径匹配。

对小白来说，先记住两句就够了：

1. 常规项目里最常用的是 `location /` 和 `location /api/`
2. 路径匹配越复杂，越要先用 `nginx -t` 和日志验证

---

## 十八、`root` 和 `alias` 的区别

这是一个非常常见的面试点，也是新手常踩的坑。

### `root`

是“拼接路径”。

```nginx
location /images/ {
    root /data/www;
}
```

请求：

```text
/images/a.png
```

实际找的文件通常是：

```text
/data/www/images/a.png
```

### `alias`

是“替换路径”。

```nginx
location /images/ {
    alias /data/static/;
}
```

请求：

```text
/images/a.png
```

实际找的文件通常是：

```text
/data/static/a.png
```

一句话区分：

- `root` 更像拼接
- `alias` 更像替换

---

## 十九、怎么让多个后端一起服务

这就是负载均衡的基础用法。

```nginx
upstream backend {
    server 10.0.0.11:8080;
    server 10.0.0.12:8080;
    server 10.0.0.13:8080;
}

server {
    listen 80;
    server_name api.example.com;

    location / {
        proxy_pass http://backend;
    }
}
```

这样用户访问时，Nginx 会把请求转发给 `backend` 里的机器。

常见扩展写法：

### 1. 权重

```nginx
upstream backend {
    server 10.0.0.11:8080 weight=3;
    server 10.0.0.12:8080 weight=1;
}
```

### 2. 失败控制

```nginx
upstream backend {
    server 10.0.0.11:8080 max_fails=3 fail_timeout=30s;
}
```

---

## 二十、HTTPS 怎么配置

现在正式环境基本都会上 HTTPS。

最基础示例：

```nginx
server {
    listen 443 ssl;
    server_name example.com;

    ssl_certificate     /etc/nginx/ssl/example.crt;
    ssl_certificate_key /etc/nginx/ssl/example.key;

    location / {
        root /data/www/example;
        index index.html;
    }
}
```

把 HTTP 跳到 HTTPS 的常见写法：

```nginx
server {
    listen 80;
    server_name example.com;
    return 301 https://$host$request_uri;
}
```

这两段通常会一起用。

---

## 二十一、文件上传大小怎么控制

很多项目一上来就会遇到这个问题。

比如上传图片、视频、附件时，如果文件太大，可能报错：

```text
413 Request Entity Too Large
```

解决方式：

```nginx
server {
    listen 80;
    server_name upload.example.com;

    location /upload/ {
        client_max_body_size 50m;
        proxy_pass http://127.0.0.1:8080;
    }
}
```

如果整个站点都想放宽，也可以直接写在 `server` 或 `http` 级别。

但更推荐按接口精确控制。

---

## 二十二、缓存怎么理解

Nginx 也经常用来做静态资源缓存控制。

比如：

```nginx
location /static/ {
    root /data/www;
    expires 7d;
}
```

意思是告诉浏览器：

这个目录下的资源可以缓存 7 天。

适合：

- 图片
- CSS
- JS
- 字体文件

但如果你的文件经常变，缓存策略就要更谨慎。

---

## 二十三、访问日志和错误日志怎么用

新手经常忽略日志，这是很可惜的。

### 1. 访问日志

看谁访问了什么、状态码是多少。

常见是：

```text
access.log
```

### 2. 错误日志

看 Nginx 自己报了什么错。

常见是：

```text
error.log
```

### 3. 出问题时怎么排

推荐顺序：

1. 先 `nginx -t`
2. 再看 `error.log`
3. 再看 `access.log`
4. 再看后端服务本身日志

---

## 二十四、修改配置后怎么生效

这一步很多人第一次会忘。

改完配置后，推荐顺序是：

### 1. 先检查语法

```bash
nginx -t
```

### 2. 再重载

```bash
nginx -s reload
```

Linux 也可以：

```bash
systemctl reload nginx
```

不要一上来就盲目重启，先检查语法更稳。

---

## 二十五、常见报错和解决思路

这一节很重要，基本是日常会遇到的问题。

### 1. 端口被占用

现象：

- 启动失败
- 访问不到

原因：

- 80 端口或你配置的端口被别的程序占用了

解决：

1. 换端口
2. 关掉占用该端口的程序

比如改成：

```nginx
listen 8081;
```

---

### 2. 403 Forbidden

现象：

- 能访问到 Nginx
- 但返回 403

常见原因：

- 没有首页文件
- 目录权限不对
- `root` 指向错误
- 目录禁止访问

排查方向：

1. 看目录里是否有 `index.html`
2. 看 `root` 路径是不是写对了
3. 看文件权限是否允许 Nginx 读取

---

### 3. 404 Not Found

现象：

- 页面找不到
- 接口找不到

常见原因：

- URL 写错
- `location` 没匹配上
- 文件不存在
- 前端路由缺少 `try_files`

排查方向：

1. 先看请求路径
2. 再看对应 `location`
3. 再看文件或后端路径是否存在

---

### 4. 500 Internal Server Error

现象：

- 服务端内部错误

很多时候，这类错误不一定是 Nginx 自己造成的，也可能是后端程序报错。

先看：

- `error.log`
- 后端程序日志

---

### 5. 502 Bad Gateway

这是最常见的问题之一。

含义通常是：

**Nginx 想把请求转给后端，但后端没正常接住。**

常见原因：

- 后端没启动
- `proxy_pass` 地址写错
- 后端端口没开
- 后端崩了

排查方向：

1. 后端服务是否正常运行
2. `proxy_pass` 地址是否正确
3. 目标端口是否可访问

---

### 6. 504 Gateway Timeout

含义通常是：

Nginx 等后端响应太久，超时了。

常见原因：

- 后端处理太慢
- 查询太重
- 网络不通畅
- 超时配置太小

可以考虑：

```nginx
proxy_read_timeout 300;
```

但注意，调大超时只是缓解，不一定是根治。

---

### 7. 413 Request Entity Too Large

含义：

上传文件太大，超过限制。

解决：

```nginx
client_max_body_size 100m;
```

---

### 8. SSL 相关报错

常见原因：

- 证书路径写错
- 私钥不匹配
- 证书已过期
- 域名和证书不对应

排查时先看：

- 证书文件是否存在
- 配置路径是否正确
- 域名是否匹配证书

---

## 二十六、常用操作清单

这一节适合收藏。

### 日常改配置

1. 打开配置文件
2. 修改内容
3. 执行 `nginx -t`
4. 执行 `nginx -s reload`
5. 浏览器验证
6. 有问题先看日志

### 部署前端页面

1. 把打包后的文件放到站点目录
2. 配置 `root`
3. 配置 `index`
4. 如果是单页应用，加 `try_files`
5. 检查并重载

### 接入后端接口

1. 确定后端地址和端口
2. 写 `location /api/`
3. 配 `proxy_pass`
4. 加常见代理头
5. 检查并重载

### 上线 HTTPS

1. 准备证书和私钥
2. 配置 `listen 443 ssl`
3. 配 `ssl_certificate` 和 `ssl_certificate_key`
4. 配置 HTTP 跳 HTTPS
5. 检查并重载

---

## 二十七、进阶使用可以学什么

如果你已经能部署静态页面、做反向代理，下一步可以学这些。

### 1. 更深入的 `location` 匹配规则

这会直接影响请求到底走哪条配置。

### 2. 负载均衡策略

比如：

- 轮询
- 权重
- 故障转移
- IP 哈希

### 3. HTTPS 优化

比如：

- 证书管理
- 强制 HTTPS
- HSTS

### 4. 缓存策略

比如：

- 静态资源缓存
- 反向代理缓存

### 5. 限流和防刷

比如：

- 单 IP 限流
- 请求频率控制

### 6. 灰度发布和多环境接入

比如：

- 开发环境
- 测试环境
- 生产环境

### 7. 配置拆分和复用

大型项目不会把所有东西都堆在一个 `nginx.conf` 里。

通常会拆成多个配置文件维护。

---

## 二十八、给你一个更像实际项目的示例

```nginx
upstream app_backend {
    server 127.0.0.1:8080;
    server 127.0.0.1:8081;
}

server {
    listen 80;
    server_name example.com www.example.com;

    location /api/ {
        proxy_pass http://app_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_read_timeout 60;
    }

    location /static/ {
        root /data/www/app;
        expires 7d;
    }

    location / {
        root /data/www/app;
        index index.html;
        try_files $uri $uri/ /index.html;
    }
}
```

这份配置同时做了几件事：

1. `/api/` 走后端集群
2. `/static/` 做静态资源缓存
3. `/` 返回前端页面
4. 支持前端单页路由

这已经是很多项目的基础雏形了。

---

## 二十九、初学者最容易踩的坑

### 1. 改完配置不 reload

文件改了，不代表生效了。

### 2. 不先 `nginx -t`

语法错了，后面全白忙。

### 3. `root` 路径写错

这是 403、404 的常见原因。

### 4. 看到 502 只盯着 Nginx

很多时候后端服务才是问题根源。

### 5. 单页应用没配 `try_files`

刷新页面容易直接 404。

### 6. 把 `root` 和 `alias` 用混

这会造成静态资源路径很诡异。

### 7. 上传限制没单独配

结果大文件上传总失败。

---

## 三十、学习 Nginx 最省事的记忆法

如果你现在觉得内容有点多，先别急着全背。

先只记住下面这 10 个关键词：

1. `server`
2. `listen`
3. `server_name`
4. `location`
5. `root`
6. `index`
7. `proxy_pass`
8. `try_files`
9. `client_max_body_size`
10. `nginx -t` / `nginx -s reload`

只要这几个概念先站稳，后面就好学很多。

---

## 三十一、最后总结

把整篇文章压缩成最核心的话，就是下面这些：

1. Nginx 是网站入口，也是流量调度员
2. 它最常见的用途是静态资源服务、反向代理、负载均衡、HTTPS 接入
3. 小白先学会部署静态网站和代理后端接口就够用了
4. 配置里最常见的核心项是 `server`、`location`、`root`、`proxy_pass`
5. 改配置后先 `nginx -t`，再 `nginx -s reload`
6. 遇到问题优先看日志，不要只靠猜
7. 进阶方向主要是 HTTPS、负载均衡、缓存、限流、配置拆分

如果你现在是第一次学 Nginx，最合适的顺序是：

1. 先跑通一个静态页面
2. 再跑通一个 `/api/` 反向代理
3. 再学 `try_files`
4. 再学 HTTPS
5. 最后再学负载均衡和限流

这样学，最不容易乱。
