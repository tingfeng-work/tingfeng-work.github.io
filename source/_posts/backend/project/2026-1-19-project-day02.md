---
title: 后端环境搭建与核心模块实战复盘
date: 2026-01-19
categories:
  - 项目
tags:
  - 餐饮管理系统
toc: true
toc_number: true
---

> 本文以一个**前后端分离的 Spring Boot 后端项目**为背景，系统梳理了从**Nginx 反向代理与负载均衡、登录鉴权、接口文档、员工模块开发**到**异常处理、拦截器、ThreadLocal、分页查询**等高频后端知识点

------

## 一、Nginx：反向代理与负载均衡

### 1. 为什么前端请求能“自动”到后端？

前端请求地址：

```
http://localhost/api/employee/login
```

后端真实接口：

```
http://localhost:8080/admin/employee/login
```

**中间桥梁：Nginx 反向代理**

### 2. 什么是反向代理？

> 反向代理是由服务器端统一接收客户端请求，再转发给内部真实服务，对客户端屏蔽后端细节。

**工程层面的价值**：

- **统一入口**：前端永远只访问 Nginx
- **解耦前后端端口与路径**
- **安全隔离**：后端服务不直接暴露
- **性能优化**：支持缓存、压缩、限流

### 3. proxy_pass 核心配置理解

```
server {
    listen 80;
    server_name localhost;

    location /api/ {
        proxy_pass http://localhost:8080/admin/;
    }
}
```

#### `/api/` + `/admin/` 是怎么拼接的？

核心规则

> **当 `location` 和 `proxy_pass` 都以 `/` 结尾时：**
>
>  **Nginx 会用 `proxy_pass` 中的路径，替换掉 `location` 匹配到的那一段路径**

**请求 URL：**

```
/api/employee/login
```

**location 命中部分：**

```
/api/
```

**剩余路径：**

```
employee/login
```

**proxy_pass：**

```
http://localhost:8080/admin/
```

**最终转发到后端的真实请求：**

```
http://localhost:8080/admin/employee/login
```

**不是拼接，而是「替换 + 保留剩余路径」**

> *“Nginx 是怎么知道 employee/login 要保留的？”*
>  location 匹配到的部分会被去掉，没有匹配到的剩余路径拼接到 proxy_pass 的路径后。

------

#### `proxy_pass` 结尾是否带 `/` 的区别

情况 1：`proxy_pass` **带 `/`**

```
location /api/ {
    proxy_pass http://localhost:8080/admin/;
}
```

**替换路径**

```
/api/xxx  →  /admin/xxx
```

**最常用、最推荐**

------------

情况 2：`proxy_pass` **不带 `/`**

```
location /api/ {
    proxy_pass http://localhost:8080/admin;
}
```

**直接拼接完整 URL**

```
/api/employee/login
↓
http://localhost:8080/admin/api/employee/login
```

小结

| proxy_pass 写法   | 转发结果                    |
| ----------------- | --------------------------- |
| `http://x/admin/` | `/api/xxx → /admin/xxx`     |
| `http://x/admin`  | `/api/xxx → /admin/api/xxx` |

------

#### Nginx 是七层代理还是四层代理？

> **这里是 HTTP 七层代理**

为什么是七层？

当前使用的是：

```
proxy_pass http://...
```

并且：

- Nginx 能看到：
  
  URL 路径（`/api/employee/login`）
  
  HTTP 方法（GET / POST）
  
  Header
  
  Cookie
- 能基于 **URI / Header / Host** 做路由

**这说明它工作在 HTTP 层（应用层）**

四层代理是什么样？

如果是四层代理（TCP）：

```
stream {
    server {
        listen 3306;
        proxy_pass mysql_backend;
    }
}
```

- 不关心 HTTP
- 不解析 URL
- 只转发 TCP 流量

------

### 总结

| 维度             | 四层代理      | 七层代理 |
| ---------------- | ------------- | -------- |
| 协议             | TCP / UDP     | HTTP     |
| 是否解析 URL     | ❌             | ✅        |
| 是否支持路径路由 | ❌             | ✅        |
| 典型场景         | MySQL / Redis | Web 接口 |

------

## 二、Nginx 负载均衡

### 本质认知

> **负载均衡 = 基于反向代理的请求分发策略**

```
upstream webservers {
    server 192.168.100.128:8080;
    server 192.168.100.129:8080;
}

location /api/ {
    proxy_pass http://webservers/admin;
}
```

### 常见负载均衡策略

| 策略       | 说明       | 典型场景         |
| ---------- | ---------- | ---------------- |
| 轮询       | 默认       | 大多数无状态接口 |
| weight     | 权重分配   | 新老机器混用     |
| ip_hash    | 固定客户端 | 需要会话一致性   |
| least_conn | 最少连接   | 长连接、慢接口   |
| url_hash   | URL 一致   | 缓存友好         |
| fair       | 响应时间   | 非官方模块       |

**注意**：
 `ip_hash` **并不能解决分布式 Session 问题**，只能缓解。

------

## 三、登录功能设计（JWT 是重点）

### 为什么不用 Session？

> HTTP 无状态 + 分布式部署的情况下：Session 粘性差 （），所以 **JWT 更适合前后端分离**

### 登录整体流程

1. Controller 接收 DTO
2. Service 校验用户：用户是否存在、密码是否匹配、状态是否禁用
3. 生成 JWT → 返回给前端
4. 前端后续请求在 Header 中携带 token

### JWT 核心组成

- **Header**：算法
- **Payload**：用户信息（如 employeeId）
- **Signature**：密钥签名
- **关键属性**：`secretKey`、`expireTime`

**注意**：

> JWT 的 Payload 只是经过 Base64URL 编码而非加密，任何人都可以解码查看其中内容，因此不应在载荷中存放密码、身份证号等敏感信息，仅保留必要的业务标识字段。

------

## 四、密码安全

### 为什么不能明文存储？

- 数据库泄露 = 全员密码泄露
- 不符合基本安全规范

### 当前方案

- 注册 / 新增员工：**MD5 加密后存储**
- 登录：明文 → MD5 → 对比

**后续可拓展**：

- MD5 不安全 → BCrypt / Argon2
- 是否加盐？在对用户密码做哈希之前，先拼接一段随机字符串（salt），再进行哈希（TODO）。
- 登录失败次数限制

------

## 五、接口文档体系（Swagger vs YApi）

### 两者定位区分（高频）

| 工具              | 阶段     | 用途           |
| ----------------- | -------- | -------------- |
| YApi              | 设计阶段 | 接口设计、评审 |
| Swagger / Knife4j | 开发阶段 | 接口调试、联调 |

> YApi 管设计，Swagger 管实现。

### Knife4j 配置要点

- Docket Bean
- Controller 扫描路径
- 静态资源映射

------

## 六、员工模块开发（CRUD 模板）

### 通用开发流程

1. 接口设计（路径 / 方法 / 参数）
2. DTO 设计
3. Controller
4. Service 接口
5. Service 实现
6. Mapper + XML
7. 测试

------

## 七、JWT 拦截器（高频 + 易追问）

### 拦截器职责

- 获取 token
- 校验 token
- 解析用户 ID
- 放行 / 拒绝

### preHandle 核心逻辑

```
String token = request.getHeader("token");
Claims claims = JwtUtil.parseJWT(secretKey, token);
Long empId = Long.valueOf(claims.get("empId").toString());
```

- token 过期会发生什么？解析失败，不会返回 claims，直接抛异常，视为 token 校验失败
- 401 vs 403 区别？401 未认证/认证失败；403 已认证但无权限
- 拦截器 vs 过滤器？过滤器是 Servlet 规范，拦截器是 Spring MVC 机制，拦截器更贴近 Controller。

| 维度                | 过滤器 Filter    | 拦截器 Interceptor |
| ------------------- | ---------------- | ------------------ |
| 所属体系            | Servlet          | Spring MVC         |
| 拦截范围            | 所有请求         | Controller 方法    |
| 能否拿到 Controller | ❌                | ✅（HandlerMethod） |
| 执行时机            | 更早             | 稍晚               |
| 典型用途            | 编码、日志、跨域 | 登录校验、权限     |

------

## 八、ThreadLocal

> ThreadLocal 为**每个线程**提供独立变量副本，避免并发冲突。

### 在本项目中的作用

- 拦截器中存用户 ID
- Service / Mapper 统一获取
- 自动填充：create_user 、update_user

**拓展**：

- ThreadLocal 内存泄漏？线程池复用，同时没有及时删除 ThreadLocal 中的值
- 为什么要 remove？**避免线程复用时脏数据串请求**，同时防止线程长期持有 value 导致OOM。
- 在 Tomcat 线程池中的问题？Tomcat 使用线程池复用线程，ThreadLocal 的生命周期会被“线程生命周期”拉长，如果不清理会导致**数据跨请求污染**和**内存滞留**。

------

## 九、全局异常处理

### 为什么要全局异常处理？

- Controller 不关心异常细节
- 统一返回格式
- 防止异常信息直接暴露

### 重复用户名异常处理

```
@ExceptionHandler(SQLIntegrityConstraintViolationException.class)
```

------

## 十、分页查询（PageHelper）

### PageHelper 核心思想

> **拦截 SQL → 自动拼接 limit**

```
PageHelper.startPage(page, pageSize);
```

### 返回结构设计

```
total + records
```

**拓展：**

- PageHelper 原理？通过拦截 SQL 执行，在原 SQL 基础上自动拼接分页语句（limit / count），并利用 ThreadLocal 传递分页参数。（count 用于返回总记录数）
- 与 MyBatis Plus 区别？PageHelper 是 MyBatis 插件、对 SQL 无侵入；MyBatis Plus 是 ORM（Object–Relational Mapping，对象关系映射：把数据库表映射为程序中的对象、把 SQL 操作映射为对象方法调用的技术。） 增强框架，分页是其内置能力。
- 大数据量分页优化？利用子查询+索引覆盖/游标/延迟关联

------

## 十一、统一更新时间格式（细节加分）

- 扩展 Spring MVC 消息转换器
- 全局 Date / LocalDateTime 处理
- 避免前后端时间不一致问题

------

## 十二、分类模块删除约束

> **不能删除仍被引用的数据**

- 分类 → 菜品 / 套餐
- 删除前先校验
- 有关联 → 抛业务异常

------

## 总结

这个项目完整覆盖了**后端环境搭建、网关层设计、鉴权体系、异常体系、分页机制与业务约束设计**，不仅实现功能，更体现了**真实生产级后端的工程思维**。

------

## 学习资料与完整代码

**已整理并上传至 GitHub 仓库**

