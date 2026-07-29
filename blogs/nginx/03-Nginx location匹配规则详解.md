---
date: 2026-07-10
is_published: true
title: Nginx location匹配规则详解
tags:
  - Nginx
categories:
  - 服务器
---

# Nginx location 匹配规则详解

很多人第一次学 Nginx 时，真正卡住的不是安装，也不是启动，而是：

**“为什么这个请求没有走我写的那条 `location`？”**

这篇文章就专门解决这个问题。

目标很明确：

1. 讲清楚 `location` 是什么
2. 讲清楚几种常见写法分别是什么意思
3. 讲清楚匹配优先级
4. 讲清楚为什么你明明写了配置，却没有生效
5. 讲清楚常见误区和排查方法

如果你看完以后，已经能自己判断“某个请求会走哪条规则”，那这篇就算达到目的了。

---

## 一、先说结论：`location` 是干什么的

在 Nginx 里，`location` 是用来匹配请求路径的。

说得再白一点：

**当请求进入某个 `server` 之后，Nginx 要靠 `location` 来决定这个路径该怎么处理。**

比如：

- `/` 返回首页
- `/api/` 转发给后端接口
- `/images/` 返回图片目录
- `/admin/` 走管理后台

这些几乎都是靠 `location` 决定的。

---

## 二、先看一个最常见的例子

```nginx
server {
    listen 80;
    server_name example.com;

    location / {
        root /data/www/app;
        index index.html;
    }

    location /api/ {
        proxy_pass http://127.0.0.1:8080;
    }
}
```

这个配置的意思很简单：

- 访问 `/api/xxx`，走后端接口
- 访问其他路径，走前端页面

也就是说，`location` 本质上就是“路径分流规则”。

---

## 三、`location` 常见写法有哪些

最常见的有下面几类。

### 1. 普通前缀匹配

```nginx
location /api/ {
    ...
}
```

意思是：

只要请求路径以 `/api/` 开头，就可能匹配到这里。

比如这些请求通常都能命中：

- `/api/user`
- `/api/login`
- `/api/order/list`

这是最常用的一种。

---

### 2. 根路径匹配

```nginx
location / {
    ...
}
```

它的意思不是“只匹配首页”，而是：

**匹配所有以 `/` 开头的路径。**

而几乎所有正常 URL 都是以 `/` 开头的，所以它经常被当作兜底规则。

比如：

- `/`
- `/about`
- `/user/info`
- `/static/a.js`

都可能匹配它。

所以很多人会把它理解成：

**默认规则。**

---

### 3. 精确匹配

```nginx
location = /login {
    ...
}
```

意思是：

只有当请求路径**完全等于** `/login` 时，才会命中。

下面这个会命中：

- `/login`

下面这些不会命中：

- `/login/`
- `/login?id=1`
- `/login/abc`

注意一点：

查询参数不参与 `location` 路径匹配的主体判断，但路径本身必须完全一致。

精确匹配很适合这些场景：

- 单独放开某个上传接口
- 单独控制某个登录接口
- 单独拦截某个固定路径

---

### 4. `^~` 前缀匹配

```nginx
location ^~ /static/ {
    ...
}
```

它也是前缀匹配，但多了一层意思：

**如果这个前缀匹配成功了，就不要再继续尝试正则匹配。**

这个点很多人第一次看会有点绕。

你先记一句够用的话：

`^~` 常用于“我非常确定这个前缀该按这条规则走，不想再被正则抢走”。

---

### 5. 区分大小写的正则匹配

```nginx
location ~ \.php$ {
    ...
}
```

意思是：

按正则规则匹配，并且区分大小写。

比如：

- `a.php` 可能匹配
- `A.PHP` 不一定匹配

正则匹配适合更灵活的路径判断，比如：

- 匹配文件后缀
- 匹配特定格式路径

---

### 6. 不区分大小写的正则匹配

```nginx
location ~* \.(jpg|png|gif)$ {
    ...
}
```

意思是：

按正则匹配，并且不区分大小写。

比如：

- `a.jpg`
- `a.JPG`
- `a.Png`

都可能匹配。

这种写法常用于静态资源后缀规则。

---

## 四、最重要的一节：匹配优先级

这部分是整篇最关键的。

很多“明明写了但不生效”的问题，本质上都是没搞清优先级。

你可以先记住这个顺序：

1. 精确匹配：`location = ...`
2. `^~` 前缀匹配
3. 正则匹配：`~` 和 `~*`
4. 普通前缀匹配，取最长前缀
5. `location /` 这种通常作为最后兜底

如果你只记一句，那就记这句：

**精确匹配最优先，普通前缀里“谁更长谁优先”，正则会插在中间参与竞争。**

不过这句话还不够稳，下面分开说。

---

## 五、先看精确匹配为什么优先级最高

```nginx
location = /api/upload {
    client_max_body_size 100m;
}

location /api/ {
    client_max_body_size 5m;
}
```

请求：

```text
/api/upload
```

会走哪条？

答案是：

```nginx
location = /api/upload
```

因为它是精确匹配，优先级最高。

这也是为什么很多上传接口、登录接口喜欢单独写一条精确规则。

---

## 六、普通前缀匹配时，谁更长谁优先

```nginx
location / {
    ...
}

location /api/ {
    ...
}

location /api/admin/ {
    ...
}
```

请求：

```text
/api/admin/user/list
```

会走哪条？

答案是：

```nginx
location /api/admin/
```

因为它前缀更长、更具体。

所以普通前缀匹配你可以这样理解：

**不是谁先写谁生效，而是谁匹配得更具体谁优先。**

这点非常重要。

---

## 七、正则匹配什么时候会参与

看这个例子：

```nginx
location /images/ {
    root /data/www;
}

location ~* \.(jpg|png|gif)$ {
    expires 7d;
}
```

请求：

```text
/images/a.jpg
```

这时你就要小心了。

因为它既能匹配前缀：

```nginx
location /images/
```

也能匹配正则：

```nginx
location ~* \.(jpg|png|gif)$
```

如果没有 `^~`，正则规则就可能参与竞争并命中。

这也是为什么有时你以为资源应该走目录规则，结果却走了文件后缀规则。

---

## 八、`^~` 到底解决了什么

还是上面的例子。

如果你写成：

```nginx
location ^~ /images/ {
    root /data/www;
}

location ~* \.(jpg|png|gif)$ {
    expires 7d;
}
```

这时请求：

```text
/images/a.jpg
```

会优先走：

```nginx
location ^~ /images/
```

不会再继续尝试正则。

所以 `^~` 的核心作用不是“更强前缀”，而是：

**前缀一旦命中，就别让正则再来抢。**

---

## 九、一个非常常见的误区：以为 `location /` 只匹配首页

很多新手会误以为：

```nginx
location / {
    ...
}
```

只处理 `/` 这个路径。

其实不是。

它几乎能匹配所有正常请求路径，所以经常被当作默认规则。

比如：

- `/`
- `/about`
- `/news/detail`
- `/abc/def`

都可能走它。

所以如果你发现“怎么所有请求都进了 `location /`”，大概率不是它有问题，而是你别的规则没有更具体地匹配上。

---

## 十、一个非常常见的误区：以为前缀匹配按书写顺序决定

看这个例子：

```nginx
location / {
    ...
}

location /api/ {
    ...
}
```

有些人会以为：

因为 `location /` 写在前面，所以 `/api/user` 会先进第一个。

不是这样。

对于普通前缀匹配，Nginx 不是按你写的先后顺序来选，而是按：

**谁匹配得更长、更具体。**

所以：

```text
/api/user
```

通常会走：

```nginx
location /api/
```

而不是 `location /`。

---

## 十一、一个非常常见的误区：以为正则一定比前缀优先

也不能这么简单理解。

看两个情况。

### 情况 1：普通前缀 + 正则

```nginx
location /static/ {
    ...
}

location ~* \.js$ {
    ...
}
```

请求：

```text
/static/app.js
```

正则有机会参与匹配。

### 情况 2：`^~` 前缀 + 正则

```nginx
location ^~ /static/ {
    ...
}

location ~* \.js$ {
    ...
}
```

请求：

```text
/static/app.js
```

这时通常优先走：

```nginx
location ^~ /static/
```

正则不会再参与。

所以真正要记住的是：

**普通前缀可能会被正则影响，`^~` 前缀会拦住正则。**

---

## 十二、怎么判断一个请求到底走哪条规则

我给你一个最实用的方法。

面对一组 `location` 时，按这个顺序想：

1. 有没有精确匹配 `=`
2. 有没有匹配成功的 `^~` 前缀
3. 有没有能命中的正则
4. 普通前缀里哪个最长
5. 都不特殊的话，最后常常落到 `location /`

你只要照这个顺序过一遍，大多数情况都能判断对。

---

## 十三、通过案例练习一下

下面直接来几组判断。

---

### 案例 1

配置：

```nginx
location / {
    return 200 "root";
}

location /api/ {
    return 200 "api";
}
```

请求：

```text
/api/user
```

结果：

```text
api
```

原因：

`/api/` 比 `/` 更长、更具体。

---

### 案例 2

配置：

```nginx
location = /api/login {
    return 200 "login";
}

location /api/ {
    return 200 "api";
}
```

请求：

```text
/api/login
```

结果：

```text
login
```

原因：

精确匹配优先级最高。

---

### 案例 3

配置：

```nginx
location /download/ {
    return 200 "download";
}

location ~* \.(zip|rar)$ {
    return 200 "file";
}
```

请求：

```text
/download/demo.zip
```

结果：

有可能走正则那条。

原因：

普通前缀命中后，正则仍可能参与。

---

### 案例 4

配置：

```nginx
location ^~ /download/ {
    return 200 "download";
}

location ~* \.(zip|rar)$ {
    return 200 "file";
}
```

请求：

```text
/download/demo.zip
```

结果：

```text
download
```

原因：

`^~` 拦住了后续正则匹配。

---

### 案例 5

配置：

```nginx
location / {
    return 200 "root";
}

location /user/ {
    return 200 "user";
}

location /user/admin/ {
    return 200 "admin";
}
```

请求：

```text
/user/admin/list
```

结果：

```text
admin
```

原因：

普通前缀中，最长匹配优先。

---

## 十四、项目里最常见的几种 `location` 组合

这部分你可以直接拿去套脑图。

### 1. 前后端分离

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:8080;
}

location / {
    root /data/www/app;
    try_files $uri $uri/ /index.html;
}
```

理解：

- `/api/` 给后端
- 其他路径给前端

---

### 2. 上传接口单独放开

```nginx
location = /api/upload {
    client_max_body_size 100m;
    proxy_pass http://127.0.0.1:8080;
}

location /api/ {
    client_max_body_size 5m;
    proxy_pass http://127.0.0.1:8080;
}
```

理解：

- 上传接口单独精确匹配
- 其他接口仍按普通限制处理

---

### 3. 静态目录优先

```nginx
location ^~ /static/ {
    root /data/www/app;
}

location ~* \.(js|css|png|jpg)$ {
    expires 7d;
}
```

理解：

- `/static/` 目录想按目录规则稳定处理
- 不想被正则后缀规则抢走

---

### 4. 固定页面单独处理

```nginx
location = /robots.txt {
    root /data/www/public;
}

location = /favicon.ico {
    root /data/www/public;
}
```

理解：

固定文件直接精确匹配最省事。

---

## 十五、为什么你写了 `location` 却没生效

最常见就这几类原因。

### 1. 请求根本没进这个 `server`

先别急着看 `location`，要先确认：

- 域名是不是进了这个 `server_name`
- 端口是不是进了这个 `listen`

如果请求连这个 `server` 都没进，里面的 `location` 再对也没用。

---

### 2. 被更高优先级的规则抢走了

比如：

- 被 `location =` 抢走
- 被 `^~` 抢走
- 被正则规则抢走
- 被更长前缀抢走

---

### 3. 路径写得不够精确

比如你以为：

```nginx
location /api {
    ...
}
```

和：

```nginx
location /api/ {
    ...
}
```

差不多。

但很多时候它们会影响实际匹配结果和路径边界，最好按你的真实需求写清楚。

---

### 4. 你以为是匹配问题，其实是后端或文件问题

比如：

- 规则匹配到了，但文件不存在
- 规则匹配到了，但 `proxy_pass` 地址错了
- 规则匹配到了，但后端没启动

这时表面看像 `location` 不生效，本质上不是。

---

## 十六、怎么排查 `location` 问题

如果你怀疑匹配错了，可以按这个顺序查。

### 1. 先确认请求访问的域名和端口

先确定是不是进了对的 `server`。

### 2. 把所有可能命中的 `location` 列出来

不要只盯着你想要的那条。

### 3. 按优先级顺序手工判断一遍

看有没有：

- `=`
- `^~`
- `~` / `~*`
- 更长的普通前缀

### 4. 看日志

尤其是：

- `access.log`
- `error.log`

### 5. 必要时临时写一个返回值测试

比如你怀疑某条规则没命中，可以临时写：

```nginx
location /api/ {
    return 200 "api";
}
```

这样一测就知道请求到底走没走进去。

这个方法很土，但非常好用。

---

## 十七、给小白的简化记忆版

如果你现在不想背太多，先记这个简化版：

1. `location = /xxx`：完全相等才匹配，优先级最高
2. `location ^~ /xxx/`：前缀匹配，命中后不再看正则
3. `location ~` / `~*`：正则匹配
4. `location /xxx/`：普通前缀匹配，谁更长谁优先
5. `location /`：通常兜底

然后再记一句：

**多数业务项目里，最常用的是 `location /`、`location /api/`、`location = /upload`。**

---

## 十八、最后总结

把整篇压缩成最核心的话，就是：

1. `location` 是 Nginx 里负责路径分流的规则
2. 常见写法有：`=`、普通前缀、`^~`、`~`、`~*`
3. 精确匹配优先级最高
4. 普通前缀不是按书写顺序，而是按最长匹配
5. 正则可能会影响普通前缀结果
6. `^~` 的作用是前缀命中后不再让正则参与
7. `location /` 不是只匹配首页，而是常见兜底规则

如果你已经能自己回答下面这几个问题，这篇就算学到了：

1. 为什么 `/api/login` 没走 `/api/`
2. 为什么图片请求被正则规则抢走了
3. 为什么 `location /` 看起来什么都能匹配
4. 为什么单独上传接口最好用精确匹配

再往下接着写，这个系列最自然的下一篇会是：

- `04-Nginx root与alias区别详解`
- 或者 `04-Nginx常见报错排查手册`

这两篇都很适合继续往实战方向走。
