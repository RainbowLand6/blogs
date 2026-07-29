---
date: 2026-07-16
is_published: true
title: 定义公共接口与 DTO
tags: [Dubbo, Java, API, DTO]
categories: [Dubbo]
---

# 定义公共接口与 DTO

目标：在 `dubbo-api` 中定义用户查询接口，并让传输对象可序列化。

## 1. 定义 DTO

`UserProfile.java`：

```java
package com.example.dubbo.api;

import java.io.Serializable;

public class UserProfile implements Serializable {
    private Long id;
    private String nickname;
    private String level;

    public UserProfile() {
    }

    public UserProfile(Long id, String nickname, String level) {
        this.id = id;
        this.nickname = nickname;
        this.level = level;
    }

    public Long getId() {
        return id;
    }

    public String getNickname() {
        return nickname;
    }

    public String getLevel() {
        return level;
    }
}
```

DTO 是跨服务传输的数据对象。它不应直接使用数据库实体，否则数据库字段、懒加载行为和内部信息会泄露到 RPC 契约中。

## 2. 定义服务接口

`UserService.java`：

```java
package com.example.dubbo.api;

public interface UserService {
    UserProfile getProfile(Long userId);
}
```

## 3. 接口设计原则

- 参数和返回值使用 DTO、基础类型或集合，不返回 ORM 实体。
- 写操作使用明确的请求 DTO，例如 `CreateUserRequest`。
- 为接口写清楚业务含义、异常和幂等要求。
- 避免把“通用 Map”作为长期接口模型，字段无法被编译器检查。

## 4. 版本演进

新增字段通常比删除字段安全。接口需要大改时，更稳妥的方式是新增 `UserServiceV2` 或新方法，给调用方迁移时间；不要直接修改旧方法的语义。

下一篇在用户服务中实现并暴露这个接口。
