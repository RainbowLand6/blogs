---
date: 2026-07-10
is_published: true
title: Nginx常见配置案例
tags:
  - Nginx
categories:
  - 服务器
---

# Nginx 常见配置案例

这篇文章适合接在《Nginx 从入门到进阶》后面看。

上一篇解决的是“知道 Nginx 是什么、怎么用”。  
这一篇解决的是“我现在要配一个场景，到底该怎么写”。

所以这篇尽量少讲空话，多给你能直接改、能直接套的配置案例。

---

## 一、使用这篇文章的方法

先说清楚怎么用最省事：

1. 先找到和你最接近的场景
2. 先把配置抄过去
3. 再改域名、端口、目录路径
4. 改完先执行 `nginx -t`
5. 再执行 `nginx -s reload`

如果你一上来就想自己从零拼，很容易把路径、匹配规则、代理地址写乱。

---

## 二、最小静态网站

这是最基础的案例。

适合场景：

- 一个纯静态页面
- 一个简单官网
- 本地测试 HTML 页面

配置：

```nginx
server {
    listen 80;
    server_name localhost;

    location / {
        root   html;
        index  index.html index.htm;
    }
}
```

你需要改的通常只有两处：

1. `server_name`
2. `root`

如果你的页面目录在：

```text
/data/www/site
```

那就改成：

```nginx
root /data/www/site;
```

---

## 三、单页应用站点

适合场景：

- Vue
- React
- Angular

这类项目最常见的问题是：  
首页能打开，但刷新某个页面后直接 404。

配置：

```nginx
server {
    listen 80;
    server_name demo.example.com;

    location / {
        root   /data/www/demo;
        index  index.html index.htm;
        try_files $uri $uri/ /index.html;
    }
}
```

关键点就在这句：

```nginx
try_files $uri $uri/ /index.html;
```

它的作用是：

1. 先找真实文件
2. 找不到再试目录
3. 还找不到就返回前端入口页 `index.html`

如果你是前端单页应用，这行通常不能少。

---

## 四、前后端分离项目

适合场景：

- 前端静态页面由 Nginx 提供
- 后端接口由 Java / Node.js / Python 提供

比如：

- 前端页面走 `/`
- 后端接口走 `/api/`

配置：

```nginx
server {
    listen 80;
    server_name app.example.com;

    location /api/ {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location / {
        root   /data/www/app;
        index  index.html index.htm;
        try_files $uri $uri/ /index.html;
    }
}
```

这个配置是最常见、也最实用的一类。

它同时做了两件事：

1. 把前端页面直接返回给浏览器
2. 把接口请求转发给后端服务

---

## 五、只做代理，不放静态页面

适合场景：

- 后端是现成系统
- Nginx 只负责对外入口
- 比如代理 Jenkins、MinIO、文档系统、内部工具平台

配置：

```nginx
server {
    listen 80;
    server_name tool.example.com;

    location / {
        proxy_pass http://127.0.0.1:8088;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

这个时候 Nginx 更像一个入口转发层。

---

## 六、多个域名对应多个站点

适合场景：

- 一台服务器上放多个站点
- 不同域名访问不同内容

配置：

```nginx
server {
    listen 80;
    server_name a.example.com;

    location / {
        root /data/www/site-a;
        index index.html;
    }
}

server {
    listen 80;
    server_name b.example.com;

    location / {
        root /data/www/site-b;
        index index.html;
    }
}
```

这个思路很简单：

- 不同 `server_name`
- 不同 `root`

这样一台机器就能放多个网站。

---

## 七、一个域名下挂多个系统

适合场景：

- 主站走 `/`
- 管理后台走 `/admin/`
- 文档系统走 `/docs/`

配置：

```nginx
server {
    listen 80;
    server_name example.com;

    location / {
        root /data/www/main;
        index index.html;
        try_files $uri $uri/ /index.html;
    }

    location /admin/ {
        root /data/www/admin;
        index index.html;
        try_files $uri $uri/ /index.html;
    }

    location /docs/ {
        proxy_pass http://127.0.0.1:9000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

不过这里要提醒一句：

如果前端项目部署在子路径下，比如 `/admin/`，前端本身的打包基础路径也要配对，不然静态资源路径容易错。

---

## 八、上传文件大小单独放开

适合场景：

- 普通接口限制较小
- 上传接口需要更大限制

比如：

- 普通接口限制 5MB
- 上传接口限制 100MB

配置：

```nginx
server {
    listen 80;
    server_name upload.example.com;

    location = /api/upload {
        client_max_body_size 100m;
        proxy_pass http://127.0.0.1:8080;
    }

    location /api/ {
        client_max_body_size 5m;
        proxy_pass http://127.0.0.1:8080;
    }
}
```

这里推荐用精确匹配：

```nginx
location = /api/upload
```

因为这样更稳，不容易误伤其他接口。

---

## 九、静态资源单独缓存

适合场景：

- 图片、JS、CSS 变化不频繁
- 想减少重复请求

配置：

```nginx
server {
    listen 80;
    server_name static.example.com;

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

这里的意思是：

- `/static/` 下的资源缓存 7 天
- 其他页面正常访问

这个配置对前端项目很常见。

---

## 十、图片目录映射

适合场景：

- 图片不在网站根目录里
- 想单独映射一个目录给外部访问

配置：

```nginx
server {
    listen 80;
    server_name img.example.com;

    location /images/ {
        alias /data/static/images/;
    }
}
```

当用户访问：

```text
/images/a.png
```

实际会去找：

```text
/data/static/images/a.png
```

这里用的是 `alias`，不是 `root`。

如果你把这两个写混，很容易出现路径错乱。

---

## 十一、统一错误页

适合场景：

- 不想让用户直接看到默认报错页
- 想统一 404、50x 页面风格

配置：

```nginx
server {
    listen 80;
    server_name example.com;

    root /data/www/error-demo;
    index index.html;

    error_page 404 /404.html;
    error_page 500 502 503 504 /50x.html;

    location = /404.html {
    }

    location = /50x.html {
    }

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

你只需要保证目录里真的有：

- `404.html`
- `50x.html`

---

## 十二、HTTPS 最基础配置

适合场景：

- 正式环境
- 已经有 SSL 证书和私钥

配置：

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

如果只是最基础先跑起来，这样就够了。

---

## 十三、HTTP 自动跳 HTTPS

这个一般和上一节一起配。

配置：

```nginx
server {
    listen 80;
    server_name example.com;
    return 301 https://$host$request_uri;
}
```

作用是：

- 用户访问 `http://example.com`
- 自动跳到 `https://example.com`

---

## 十四、反向代理后端集群

适合场景：

- 后端有多台机器
- 想让 Nginx 做统一入口

配置：

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
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

这是最常见的负载均衡入门写法。

---

## 十五、带权重的负载均衡

适合场景：

- 有的机器性能更强
- 想让强一点的机器多分担流量

配置：

```nginx
upstream backend {
    server 10.0.0.11:8080 weight=3;
    server 10.0.0.12:8080 weight=2;
    server 10.0.0.13:8080 weight=1;
}
```

简单理解：

- `10.0.0.11` 分到的请求更多
- `10.0.0.13` 分到的请求更少

---

## 十六、后端响应慢时放宽超时

适合场景：

- 导出报表
- 大文件处理
- 慢查询接口

配置：

```nginx
location /api/report/ {
    proxy_pass http://127.0.0.1:8080;
    proxy_connect_timeout 60;
    proxy_send_timeout 60;
    proxy_read_timeout 300;
}
```

但这里要注意：

如果只是无限放大超时，而不看后端为什么慢，通常只是拖延问题，不是解决问题。

---

## 十七、限制访问频率

这个算入门级进阶。

适合场景：

- 登录接口防刷
- 验证码接口防刷
- 某些敏感接口防恶意高频请求

配置：

```nginx
http {
    limit_req_zone $binary_remote_addr zone=login_limit:10m rate=5r/s;

    server {
        listen 80;
        server_name example.com;

        location /api/login {
            limit_req zone=login_limit burst=10 nodelay;
            proxy_pass http://127.0.0.1:8080;
        }
    }
}
```

对小白来说，你先知道它能干这个就够了。  
真正上线前再按业务情况慢慢调。

---

## 十八、屏蔽敏感路径

适合场景：

- 不想暴露某些目录
- 想禁止直接访问后台文件

配置：

```nginx
location /private/ {
    return 403;
}
```

或者：

```nginx
location ~* \.(env|log|sql)$ {
    deny all;
}
```

这个在安全上很实用。

---

## 十九、一个更接近生产环境的综合案例

```nginx
upstream app_backend {
    server 127.0.0.1:8080;
    server 127.0.0.1:8081;
}

server {
    listen 80;
    server_name example.com www.example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name example.com www.example.com;

    ssl_certificate     /etc/nginx/ssl/example.crt;
    ssl_certificate_key /etc/nginx/ssl/example.key;

    location = /api/upload {
        client_max_body_size 100m;
        proxy_pass http://app_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location /api/ {
        client_max_body_size 10m;
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

这份配置同时做了这些事：

1. HTTP 自动跳 HTTPS
2. HTTPS 提供正式访问
3. `/api/upload` 单独放宽上传大小
4. `/api/` 转发给后端集群
5. `/static/` 做静态资源缓存
6. `/` 提供前端页面并支持单页应用路由

这已经是一个很常见的实际项目形态了。

---

## 二十、抄配置前先确认这几项

很多人不是不会抄，而是抄完没改对地方。

你至少要核对下面这些值：

1. 域名是不是你的域名
2. 端口是不是你的端口
3. 静态目录是不是你的真实目录
4. 后端地址是不是你的真实服务地址
5. HTTPS 证书路径是不是存在
6. 上传大小是不是符合你的业务需求

如果这几项没核对，配置看起来再像，也可能完全跑不起来。

---

## 二十一、套用后常见报错怎么排

### 1. 页面打不开

先看：

- `listen` 对不对
- 域名有没有解析到服务器
- 端口有没有开放

### 2. 静态资源 404

先看：

- `root` 或 `alias` 是否写对
- 文件是否真的存在
- 子路径部署时前端打包路径是否正确

### 3. 接口 502

先看：

- 后端是否启动
- `proxy_pass` 是否写对
- 后端端口是否能通

### 4. 上传报 413

先看：

- `client_max_body_size` 是否太小

### 5. HTTPS 起不来

先看：

- 证书路径是否存在
- 私钥是否匹配
- 443 端口是否开放

---

## 二十二、最后总结

如果你现在是“会一点，但还不太稳”的阶段，这篇最值得先掌握的是下面 6 类案例：

1. 静态网站
2. 前后端分离
3. 单页应用刷新不 404
4. 上传大小单独控制
5. HTTPS 和 HTTP 跳转
6. 后端集群负载均衡

把这几类先跑通，Nginx 的日常使用你就已经覆盖大半了。

如果继续往下写，这个系列最自然的下一篇就是：

- `03-Nginx location 匹配规则详解`
- 或者 `03-Nginx 常见报错排查手册`

这两篇都会很实用。
