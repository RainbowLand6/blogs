# Nginx 配置详解与常用配置示例

## 1. 文档目标

本文档做两件事：

1. 详细解释你提供的这份 Nginx 配置文件在做什么
2. 总结 Nginx 常见配置项、作用、使用位置和示例

当前分析的配置特点是：

- 主要用于多个域名的 Web 站点和反向代理
- 统一通过 `upstream gateway` 转发到后端 Java 网关
- 多个前端站点通过不同 `server_name` 区分
- 对上传接口单独设置 `client_max_body_size`
- 同时承载静态前端页面和接口转发

---

## 2. 先看这份配置整体在干什么

这份配置本质上是一个“多站点入口网关”，主要职责有：

- 根据不同域名，把请求分发给不同站点
- 把 `/sunrise-gateway`、`/sunrise-gateway-b2c` 这类接口请求转发给后端服务
- 把 `/`、`/payui` 这类前端页面请求指向本地静态目录
- 对上传接口单独放宽请求体大小限制
- 为单页应用使用 `try_files ... /index.html`

简单说：

- 前端页面由 Nginx 直接提供
- 后端接口由 Nginx 反向代理到 `172.17.140.201:8080`

---

## 3. 配置文件结构说明

Nginx 配置通常按层级组织：

1. 全局块
2. `events` 块
3. `http` 块
4. `upstream` 块
5. `server` 块
6. `location` 块

你这份配置也完全符合这个结构。

---

## 4. 全局配置详解

### 4.1 `#user nobody;`

```nginx
#user  nobody;
```

作用：

- 指定 Nginx worker 进程以哪个系统用户身份运行

说明：

- 这一行被注释掉了，所以当前不生效
- 如果不写，通常使用编译或启动时的默认用户

示例：

```nginx
user nginx;
```

适用场景：

- 生产环境一般会显式指定为低权限用户，比如 `nginx`、`www-data`

---

### 4.2 `worker_processes 1;`

```nginx
worker_processes  1;
```

作用：

- 指定 worker 进程数量

含义：

- `1` 表示只启动 1 个 worker 进程处理请求

建议：

- 开发环境可以这样配
- 生产环境更常见的是：

```nginx
worker_processes auto;
```

这样会根据 CPU 核数自动设置

---

### 4.3 `error_log`

```nginx
#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;
```

作用：

- 指定错误日志文件和日志级别

常见级别：

- `debug`
- `info`
- `notice`
- `warn`
- `error`
- `crit`

示例：

```nginx
error_log /var/log/nginx/error.log warn;
```

说明：

- 你这里都被注释掉了，说明用的是默认错误日志配置

---

### 4.4 `pid`

```nginx
#pid        logs/nginx.pid;
```

作用：

- 指定 Nginx 主进程 PID 文件位置

用途：

- 用于管理 Nginx 进程，比如 reload、stop 时读取 PID

示例：

```nginx
pid /run/nginx.pid;
```

---

## 5. `events` 块详解

```nginx
events {
    worker_connections  1024;
}
```

### 5.1 `worker_connections 1024;`

作用：

- 每个 worker 进程允许同时打开的最大连接数

注意：

- 这不是“系统总并发”，而是“单个 worker 的连接上限”
- 理论最大连接数大致约等于：

```text
worker_processes * worker_connections
```

如果：

- `worker_processes = 1`
- `worker_connections = 1024`

那理论上大约支持 1024 个并发连接

生产常见示例：

```nginx
events {
    worker_connections 4096;
    use epoll;
}
```

---

## 6. `http` 块详解

`http` 块是最核心的 HTTP 服务配置区。你这份文件里，大多数主要配置都在这里。

```nginx
http {
    include       mime.types;
    default_type  application/octet-stream;

    client_header_buffer_size 4k;
    large_client_header_buffers 4 8k;

    client_header_timeout 300;
    client_body_timeout 300;
    proxy_read_timeout 300;

    sendfile        on;
    keepalive_timeout  65;

    upstream gateway {  
        server 172.17.140.201:8080;
    }

    ...
}
```

---

### 6.1 `include mime.types;`

```nginx
include mime.types;
```

作用：

- 加载文件扩展名与 Content-Type 的映射关系

例如：

- `.html` -> `text/html`
- `.css` -> `text/css`
- `.js` -> `application/javascript`
- `.png` -> `image/png`

没有它会怎样：

- 静态资源的响应类型可能不正确

---

### 6.2 `default_type application/octet-stream;`

```nginx
default_type  application/octet-stream;
```

作用：

- 当 Nginx 无法判断文件类型时，使用默认 MIME 类型

`application/octet-stream` 含义：

- 通用二进制流

常见用途：

- 下载文件
- 未知文件类型的兜底处理

---

### 6.3 `client_header_buffer_size 4k;`

```nginx
client_header_buffer_size 4k;
```

作用：

- 读取客户端请求头时使用的单个缓冲区大小

典型场景：

- 请求头较大
- Cookie 很长
- 自定义 Header 很多

如果太小，可能触发请求头过大问题

---

### 6.4 `large_client_header_buffers 4 8k;`

```nginx
large_client_header_buffers 4 8k;
```

作用：

- 为“大请求头”额外提供更大的缓冲区

含义：

- 最多使用 4 个缓冲区
- 每个缓冲区大小 8KB

典型用途：

- 处理超长 Cookie
- 处理较长 URL
- 处理复杂鉴权请求头

---

### 6.5 `client_header_timeout 300;`

```nginx
client_header_timeout 300;
```

作用：

- 等待客户端发送完整请求头的超时时间，单位秒

这里的值：

- `300` 秒，也就是 5 分钟

通常默认不会配这么大，说明这里对慢请求比较宽松

---

### 6.6 `client_body_timeout 300;`

```nginx
client_body_timeout 300;
```

作用：

- 等待客户端发送请求体的超时时间

适合场景：

- 文件上传
- 慢网络环境

如果上传视频、图片较大，这个配置很有意义

---

### 6.7 `proxy_read_timeout 300;`

```nginx
proxy_read_timeout 300;
```

作用：

- Nginx 等待上游服务响应的超时时间

这里的“上游”是指：

- `proxy_pass` 指向的后端服务

如果后端处理上传较慢，这个值太小可能导致：

- 504 Gateway Timeout

---

### 6.8 日志格式 `log_format` 与 `access_log`

```nginx
#log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
#                  '$status $body_bytes_sent "$http_referer" '
#                  '"$http_user_agent" "$http_x_forwarded_for"';
#access_log  logs/access.log  main;
```

作用：

- 定义访问日志输出格式
- 指定访问日志文件

说明：

- 你这里也注释掉了
- 当前可能走的是别处定义，或者使用默认日志方式

常见示例：

```nginx
log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                '$status $body_bytes_sent "$http_referer" '
                '"$http_user_agent" "$http_x_forwarded_for"';
access_log /var/log/nginx/access.log main;
```

---

### 6.9 `sendfile on;`

```nginx
sendfile on;
```

作用：

- 开启高效文件传输

效果：

- 静态文件发送性能更高
- 减少用户态和内核态之间的拷贝

典型用于：

- HTML
- CSS
- JS
- 图片
- 下载文件

---

### 6.10 `keepalive_timeout 65;`

```nginx
keepalive_timeout  65;
```

作用：

- 保持客户端连接的超时时间

意义：

- 在这个时间内复用 TCP 连接，减少反复建连成本

设置太大：

- 会占用更多连接资源

设置太小：

- 会增加频繁建连开销

65 秒是很常见的值

---

## 7. `upstream gateway` 详解

```nginx
upstream gateway {  
    server 172.17.140.201:8080;
}
```

作用：

- 定义一个上游服务组，名字叫 `gateway`

后续凡是：

```nginx
proxy_pass http://gateway;
```

都会被转发到：

```text
172.17.140.201:8080
```

这属于反向代理的标准写法。

### 7.1 当前写法含义

这里只有一个后端节点：

```nginx
server 172.17.140.201:8080;
```

所以它不是负载均衡，只是做了一个“命名转发入口”。

### 7.2 常见扩展示例

#### 多节点轮询

```nginx
upstream gateway {
    server 172.17.140.201:8080;
    server 172.17.140.202:8080;
    server 172.17.140.203:8080;
}
```

#### 权重

```nginx
upstream gateway {
    server 172.17.140.201:8080 weight=3;
    server 172.17.140.202:8080 weight=1;
}
```

#### 失败控制

```nginx
upstream gateway {
    server 172.17.140.201:8080 max_fails=3 fail_timeout=30s;
}
```

---

## 8. `server` 块整体思路

你这份配置里有很多个 `server` 块，模式基本一样：

- `listen 80;`
- `server_name 某个域名`
- `/sunrise-gateway` 转发到后端
- `/` 指向某个前端目录
- 某些站点还有 `/payui`
- 某些站点还有 `/sunrise-gateway-b2c`
- 都配置了错误页

这属于典型的“多域名、多前端、多接口入口”配置。

---

## 9. 第一类 `server`：纯反向代理工具站点

例如：

```nginx
server {
    listen       80;
    server_name  jenkins.dev1.jjj.tool;
    location / {
        proxy_pass  http://127.0.0.1:8088;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host  $host;
        proxy_set_header X-Forwarded-Port  $server_port;
    }
}
```

类似还有：

- `showdoc.dev1.jjj.tool`
- `ittools.dev1.jjj.tool`
- `1panel.dev1.jjj.tool`

### 9.1 `listen 80;`

作用：

- 监听 HTTP 80 端口

说明：

- 这些站点当前都只开了 HTTP，没有配置 HTTPS

### 9.2 `server_name`

例如：

```nginx
server_name jenkins.dev1.jjj.tool;
```

作用：

- 根据请求头中的 Host 来匹配站点

也就是：

- 请求 `jenkins.dev1.jjj.tool` 进入这个 `server`
- 请求 `showdoc.dev1.jjj.tool` 进入另一个 `server`

### 9.3 `location /`

```nginx
location / {
    proxy_pass http://127.0.0.1:8088;
    ...
}
```

作用：

- 代理当前域名下的所有路径到后端服务

### 9.4 `proxy_pass`

```nginx
proxy_pass http://127.0.0.1:8088;
```

作用：

- 把请求转发到本机 `8088` 端口服务

常见用途：

- Jenkins
- ShowDoc
- 内部工具服务

### 9.5 一组常见 `proxy_set_header`

```nginx
proxy_set_header Host              $host;
proxy_set_header X-Real-IP         $remote_addr;
proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header X-Forwarded-Host  $host;
proxy_set_header X-Forwarded-Port  $server_port;
```

这些头的作用如下：

#### `Host`

```nginx
proxy_set_header Host $host;
```

把原始请求域名传给后端。

后端可以根据它识别：

- 当前访问的是哪个域名

#### `X-Real-IP`

```nginx
proxy_set_header X-Real-IP $remote_addr;
```

把客户端真实 IP 传给后端。

#### `X-Forwarded-For`

```nginx
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```

记录完整代理链上的客户端 IP。

#### `X-Forwarded-Proto`

```nginx
proxy_set_header X-Forwarded-Proto $scheme;
```

告诉后端原始协议是：

- `http`
- 或 `https`

#### `X-Forwarded-Host`

```nginx
proxy_set_header X-Forwarded-Host $host;
```

把原始 Host 透传给后端。

#### `X-Forwarded-Port`

```nginx
proxy_set_header X-Forwarded-Port $server_port;
```

告诉后端外部入口端口。

这组头很标准，适合反向代理后端 Web 服务。

---

## 10. 第二类 `server`：前端静态站点 + 网关接口

例如：

```nginx
server {
    listen       80;
    server_name  d46adminlogin.jiujiajiu.com;

    location = /sunrise-gateway/oss/ossUpload {
        proxy_pass http://gateway;
        client_max_body_size 50m;
        proxy_set_header Host $host;
    }

    location /sunrise-gateway {
        proxy_pass http://gateway;
        root sunrise-gateway;
        client_max_body_size 5m;
        proxy_set_header Host $host;

        proxy_buffering on;
        proxy_buffer_size 4k;
        proxy_buffers 8 1M;
        proxy_busy_buffers_size 2M;
        proxy_max_temp_file_size 0;
    }

    location ^~ / {
        root   html/adminlogin;
        index  index.html index.htm;
        try_files $uri $uri/ /index.html;

        client_max_body_size 10m;

        add_header Cache-Control no-cache;
    }
}
```

这是整份配置里最重要的一类。

---

### 10.1 `location = /sunrise-gateway/oss/ossUpload`

```nginx
location = /sunrise-gateway/oss/ossUpload {
    proxy_pass http://gateway;
    client_max_body_size 50m;
    proxy_set_header Host $host;
}
```

这是一个精确匹配。

含义：

- 只有当请求路径 **完全等于**
  `/sunrise-gateway/oss/ossUpload`
  时，才命中这个 location

作用：

- 单独给上传接口放大请求体限制

这是你们这次解决视频上传 413 的关键。

#### `client_max_body_size 50m;`

作用：

- 允许请求体最大 50MB

如果上传文件超过这个大小：

- Nginx 会直接返回 `413 Request Entity Too Large`

这条配置是“只对这个接口生效”的，非常适合精准控制。

常见示例：

```nginx
location = /upload {
    client_max_body_size 200m;
    proxy_pass http://backend;
}
```

---

### 10.2 `location /sunrise-gateway`

```nginx
location /sunrise-gateway {
    proxy_pass http://gateway;
    root sunrise-gateway;
    client_max_body_size 5m;
    proxy_set_header Host $host;

    proxy_buffering on;
    proxy_buffer_size 4k;
    proxy_buffers 8 1M;
    proxy_busy_buffers_size 2M;
    proxy_max_temp_file_size 0;
}
```

作用：

- 把 `/sunrise-gateway` 开头的请求转发给后端 `gateway`

比如：

- `/sunrise-gateway/user/login`
- `/sunrise-gateway/order/list`
- `/sunrise-gateway/oss/xxx`

都会匹配这个 location

但如果有更精确的：

```nginx
location = /sunrise-gateway/oss/ossUpload
```

那么上传接口会优先走更精确那条。

#### `client_max_body_size 5m;`

作用：

- 一般接口只允许 5MB 请求体

这也是之前视频上传报 413 的根因。

#### `root sunrise-gateway;`

这一项在 `proxy_pass` 场景里通常没有实际意义，容易让人误解。

因为：

- `root` 主要用于本地静态文件查找
- `proxy_pass` 主要用于转发到上游服务

这类写法虽然能存在，但通常建议删除无关的 `root`，避免混淆。

#### `proxy_buffering on;`

作用：

- 开启代理响应缓冲

好处：

- 减少慢客户端对上游的直接拖累
- 提高整体吞吐

#### `proxy_buffer_size 4k;`

作用：

- 用于读取上游响应头的缓冲区大小

#### `proxy_buffers 8 1M;`

作用：

- 分配 8 个缓冲区，每个 1MB
- 用来缓冲上游响应体

#### `proxy_busy_buffers_size 2M;`

作用：

- 正在向客户端发送响应时，允许占用的忙碌缓冲区大小

#### `proxy_max_temp_file_size 0;`

作用：

- 禁止把过大的代理响应写入临时文件

影响：

- 响应尽量保留在内存中
- 可减少磁盘 IO
- 但会增大内存压力

---

### 10.3 `location ^~ /`

```nginx
location ^~ / {
    root   html/adminlogin;
    index  index.html index.htm;
    try_files $uri $uri/ /index.html;

    client_max_body_size 10m;

    add_header Cache-Control no-cache;
}
```

这个块主要负责前端静态页面。

#### `location ^~ /`

作用：

- 匹配以 `/` 开头的请求

这里的 `^~` 表示：

- 如果前缀匹配成功，就不再继续做正则匹配

你这份配置里没写正则 `location`，所以这里更多是显式表达“这是前端根路径处理规则”。

#### `root html/adminlogin;`

作用：

- 指定站点根目录

比如请求：

```text
/js/app.js
```

会去找本地文件：

```text
html/adminlogin/js/app.js
```

#### `index index.html index.htm;`

作用：

- 当请求目录时，默认返回首页文件

例如请求 `/` 时：

- 优先找 `index.html`
- 再找 `index.htm`

#### `try_files $uri $uri/ /index.html;`

这是单页应用很常见的写法。

作用：

1. 先尝试访问当前请求路径对应文件
2. 再尝试访问同名目录
3. 如果都不存在，返回 `/index.html`

适用于：

- Vue Router history 模式
- React Router
- Angular SPA

例子：

用户直接访问：

```text
/order/detail/123
```

Nginx 本地找不到这个真实文件时，就返回：

```text
/index.html
```

然后由前端路由接管。

#### `client_max_body_size 10m;`

作用：

- 当前站点根路径允许上传 10MB

这个值通常是给某些表单、图片上传、富文本上传兜底用的。

#### `add_header Cache-Control no-cache;`

作用：

- 告诉浏览器不要强缓存

适合：

- 频繁发版的前端项目
- 希望用户尽快拿到新版本页面

---

## 11. 第三类路径：`/payui`

例如：

```nginx
location ^~ /payui {
    root   html/payui;
    index  index.html index.htm;
    try_files $uri $uri/ /index.html;
    client_max_body_size 5m;

    add_header Cache-Control no-cache;
}
```

作用：

- 单独给支付相关前端页面提供静态资源目录

适合场景：

- 一个域名下部署多个前端子应用

例如：

- 主站点在 `html/www`
- 支付页在 `html/payui`

---

## 12. 第四类路径：`/sunrise-gateway-b2c`

例如：

```nginx
location /sunrise-gateway-b2c {
    proxy_pass http://gateway;
    root sunrise-gateway;
    client_max_body_size 5m;
    proxy_set_header Host $host;

    proxy_buffering on;
    proxy_buffer_size 4k;
    proxy_buffers 8 1M;
    proxy_busy_buffers_size 2M;
    proxy_max_temp_file_size 0;
}
```

作用：

- 转发另一组后端接口，可能是 B2C 业务接口

与 `/sunrise-gateway` 的区别：

- 路径不同
- 可能对应不同业务模块

同样的，还能看到某些站点单独对：

```nginx
location /sunrise-gateway-b2c/oss
location /sunrise-gateway-b2c/oss/ossUpload
```

做更大上传限制，这和你们给 `/sunrise-gateway/oss/ossUpload` 单独放开的思路是一样的。

---

## 13. 错误页配置

```nginx
error_page   500 502 503 504  /50x.html;
location = /50x.html {
    root   html;
}
```

作用：

- 当发生 500、502、503、504 时，返回自定义错误页

含义：

- 错误页路径是 `/50x.html`
- 它实际对应本地文件 `html/50x.html`

这能让错误页面更统一，不至于直接看到默认报错页。

---

## 14. 这份配置的设计特点

### 14.1 优点

1. 结构清楚  
   域名、前端、接口职责分开

2. 前后端分离明显  
   静态页面本地提供，接口统一转发

3. 上传接口做了单独限制  
   这比整体放大更合理

4. 单页应用兼容处理正确  
   `try_files ... /index.html` 很标准

5. 多个域名复用同一套后端网关  
   便于统一治理

### 14.2 可优化点

1. `proxy_pass` 的 location 里混用了 `root`  
   在代理场景里通常没必要，建议删掉无效 `root`

2. 大量重复配置  
   `proxy_buffering`、`proxy_set_header Host $host`、`client_max_body_size` 等可以适当抽取

3. 很多站点仍然只监听 80  
   如果是正式环境，建议补 HTTPS

4. 上传大小不够统一  
   不同站点的 `/ossUpload` 有的是 `10m`，有的是 `50m`，最好按业务需求统一标准

---

## 15. Nginx 常用配置项总表

下面把常见配置项按用途分类总结。

---

### 15.1 进程与性能类

#### `worker_processes`

作用：

- worker 进程数

示例：

```nginx
worker_processes auto;
```

#### `worker_connections`

作用：

- 每个 worker 最大连接数

示例：

```nginx
events {
    worker_connections 4096;
}
```

#### `sendfile`

作用：

- 高效传输静态文件

示例：

```nginx
sendfile on;
```

#### `keepalive_timeout`

作用：

- 长连接保持时间

示例：

```nginx
keepalive_timeout 65;
```

---

### 15.2 日志类

#### `error_log`

作用：

- 错误日志

示例：

```nginx
error_log /var/log/nginx/error.log warn;
```

#### `access_log`

作用：

- 访问日志

示例：

```nginx
access_log /var/log/nginx/access.log main;
```

#### `log_format`

作用：

- 自定义访问日志格式

示例：

```nginx
log_format main '$remote_addr - $host "$request" $status $body_bytes_sent';
```

---

### 15.3 请求大小与超时类

#### `client_max_body_size`

作用：

- 限制请求体大小

示例：

```nginx
client_max_body_size 100m;
```

典型报错：

- 超过限制时返回 `413 Request Entity Too Large`

#### `client_body_timeout`

作用：

- 读取请求体超时

示例：

```nginx
client_body_timeout 300;
```

#### `client_header_timeout`

作用：

- 读取请求头超时

示例：

```nginx
client_header_timeout 60;
```

---

### 15.4 反向代理类

#### `proxy_pass`

作用：

- 代理请求到上游

示例：

```nginx
location /api/ {
    proxy_pass http://backend;
}
```

#### `proxy_set_header`

作用：

- 向后端传递请求头

示例：

```nginx
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```

#### `proxy_read_timeout`

作用：

- 等待后端响应超时

示例：

```nginx
proxy_read_timeout 300;
```

#### `proxy_connect_timeout`

作用：

- 与后端建立连接超时

示例：

```nginx
proxy_connect_timeout 30;
```

#### `proxy_send_timeout`

作用：

- 向后端发送请求超时

示例：

```nginx
proxy_send_timeout 300;
```

#### `proxy_buffering`

作用：

- 是否缓冲后端响应

示例：

```nginx
proxy_buffering on;
```

---

### 15.5 静态站点类

#### `root`

作用：

- 指定站点根目录

示例：

```nginx
root /data/www;
```

#### `alias`

作用：

- 给某个路径映射到指定目录

示例：

```nginx
location /images/ {
    alias /data/static/images/;
}
```

和 `root` 的区别：

- `root` 是拼接请求路径
- `alias` 是替换匹配前缀

#### `index`

作用：

- 默认首页文件

示例：

```nginx
index index.html index.htm;
```

#### `try_files`

作用：

- 依次尝试查找文件，找不到则走兜底路径

示例：

```nginx
try_files $uri $uri/ /index.html;
```

---

### 15.6 匹配规则类

#### `location =`

作用：

- 精确匹配

示例：

```nginx
location = /login {
    ...
}
```

#### `location /xxx`

作用：

- 普通前缀匹配

示例：

```nginx
location /api {
    ...
}
```

#### `location ^~ /xxx`

作用：

- 前缀匹配成功后，不再继续正则匹配

示例：

```nginx
location ^~ /static/ {
    ...
}
```

#### `location ~`

作用：

- 区分大小写的正则匹配

示例：

```nginx
location ~ \.php$ {
    ...
}
```

#### `location ~*`

作用：

- 不区分大小写的正则匹配

示例：

```nginx
location ~* \.(jpg|png|gif)$ {
    expires 7d;
}
```

---

### 15.7 缓存与响应头类

#### `add_header`

作用：

- 给响应增加 Header

示例：

```nginx
add_header Cache-Control no-cache;
```

#### `expires`

作用：

- 控制浏览器缓存时间

示例：

```nginx
location /static/ {
    expires 7d;
}
```

#### `etag`

作用：

- 控制是否启用 ETag

示例：

```nginx
etag on;
```

---

### 15.8 HTTPS 类

最基本示例：

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

常见配套：

```nginx
server {
    listen 80;
    server_name example.com;
    return 301 https://$host$request_uri;
}
```

---

## 16. 几个实用示例

### 16.1 静态站点示例

```nginx
server {
    listen 80;
    server_name demo.example.com;

    location / {
        root /data/www/demo;
        index index.html;
        try_files $uri $uri/ /index.html;
    }
}
```

### 16.2 API 反向代理示例

```nginx
upstream api_backend {
    server 10.0.0.11:8080;
    server 10.0.0.12:8080;
}

server {
    listen 80;
    server_name api.example.com;

    location / {
        proxy_pass http://api_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

### 16.3 单独放开上传大小示例

```nginx
server {
    listen 80;
    server_name upload.example.com;

    location = /api/upload {
        proxy_pass http://backend;
        client_max_body_size 200m;
    }

    location /api/ {
        proxy_pass http://backend;
        client_max_body_size 5m;
    }
}
```

### 16.4 前后端分离示例

```nginx
upstream java_gateway {
    server 172.17.140.201:8080;
}

server {
    listen 80;
    server_name app.example.com;

    location /api/ {
        proxy_pass http://java_gateway;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location / {
        root /data/www/app;
        index index.html;
        try_files $uri $uri/ /index.html;
    }
}
```

---

## 17. 结合你这份配置的关键结论

### 17.1 你们这份配置的核心模式

- 多域名入口
- 前端静态资源本地部署
- 网关接口统一转发到 `gateway`
- 某些上传接口单独提高 `client_max_body_size`

### 17.2 之前上传报 413 的根因

根因就是类似这样的配置：

```nginx
location /sunrise-gateway {
    client_max_body_size 5m;
}
```

而上传视频接口如果没有单独更精确匹配，就会落到这条规则上，超过 5MB 就直接被 Nginx 拦截。

### 17.3 当前修复思路是合理的

例如：

```nginx
location = /sunrise-gateway/oss/ossUpload {
    proxy_pass http://gateway;
    client_max_body_size 50m;
}
```

这是更合理的做法，因为它：

- 只放开一个接口
- 不影响其他普通接口
- 更符合最小改动原则

---

## 18. 建议的维护方式

如果后续还会继续扩展，可以考虑：

1. 把重复的 `proxy_set_header` 提到公共片段
2. 把各站点公共代理规则拆成单独 include 文件
3. 把上传大小标准统一整理
4. 为正式域名补 HTTPS
5. 给重要接口单独配访问日志，便于排查上传问题

例如：

```nginx
location = /sunrise-gateway/oss/ossUpload {
    access_log /var/log/nginx/oss_upload_access.log main;
    proxy_pass http://gateway;
    client_max_body_size 50m;
    proxy_read_timeout 300;
}
```

---

## 19. 最后总结

这份 Nginx 配置本质上是一个：

- 多域名入口层
- 静态资源服务器
- 反向代理网关
- 上传限制控制层

如果只抓最关键的理解点，可以记住这 6 条：

1. `server_name` 决定域名进哪个站点
2. `location` 决定路径怎么处理
3. `root` 用来找本地静态文件
4. `proxy_pass` 用来转发到后端服务
5. `try_files ... /index.html` 是单页应用常见写法
6. `client_max_body_size` 决定上传大小上限，超了就是 `413`

如果后续你要继续细化，我建议下一步可以单独再整理两份：

1. “你们这份配置里所有 `location` 匹配优先级说明”
2. “Nginx 反向代理、静态站点、上传接口的最佳实践模板”

