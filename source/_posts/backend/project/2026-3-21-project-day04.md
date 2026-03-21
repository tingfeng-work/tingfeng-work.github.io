---
title: 项目Redis、小程序相关总结
date: 2026-03-21
categories:
  - 项目
tags:
  - 餐饮管理系统
toc: true
toc_number: true
---

------

# 一、Redis 

## 1. 为什么在这个场景引入 Redis？

在店铺营业状态这个场景中：

- 数据特点：只有两种状态（营业 / 打烊）、读多写少（频繁查询状态）、对实时性要求高
- 如果用 MySQL：需要建表，增加系统复杂度、每次查询需要 IO，性能较低

因此选择使用 **Redis（内存数据库）** 来存储状态：

- 基于内存，读写性能高（微秒级）
- 适合存储简单 KV 结构
- 可以减少数据库压力

------

## 2. Redis 在项目中的使用流程

1. 引入依赖：spring-boot-starter-data-redis
2. 配置连接信息：host / port / password / database
3. 配置 RedisTemplate 类：设置 key/value 序列化器（避免乱码）
4. 业务操作：set / get

------

## 3. 为什么要配置序列化器？

RedisTemplate 默认使用 JDK 序列化：可读性差（乱码）、跨语言兼容性差

在项目中：

- key：StringRedisSerializer
- value：String / JSON 序列化

好处：可读性强、便于调试、通用性好

------

## 4. 怎么设计Key？

全局设置一个常量：

```
 public static final String KEY = "SHOP_STATUS";
```

------

## 5. Bug 排查过程（重点）

你这段非常好，但需要“面试表达化”👇

问题：

```
RedisConnectionFailureException: Unable to connect to Redis
```

排查思路（一定要按层次说！）：

### （1）排除 Spring Bean 问题

- redisTemplate 成功注入（打印 redisTemplate对象）

### （2）排除 Redis 服务问题

- redis-cli 连接正常 (redis-cli)

### （3）排除配置文件问题

- host / port / database 读取正常（@value）

### （4）定位到连接工厂（关键）

- RedisConnectionFactory 创建失败 ❌

最终定位：Lettuce 客户端连接阶段失败

------

## 6. 问题本质原因

> 在 Windows 环境下，Redis 开启密码认证时，Lettuce 客户端在认证阶段存在兼容问题，导致连接失败。

------

## 7. 解决方案（要说“方案 + 扩展”）

当前方案：

- 去掉 Redis 密码 → 成功连接

更优方案（面试加分）：

- 使用 **Docker + Linux 版本 Redis**
- 或切换 Jedis

------

## 扩展问题

- Redis 和 MySQL 的区别？
- Redis 为什么快？
- Redis 持久化机制（RDB / AOF）
- Redis 线程模型（单线程 + IO 多路复用）

------

# 二、微信小程序登录 + JWT

------

## 1. 小程序登录流程

1. 小程序调用 wx.login → 获取 **code**
2. 客户端将 code 发送给后端
3. 后端调用微信接口（通过 HttpClient）
4. 微信返回：
   - openid（用户唯一标识）
5. 后端：
   - 根据 openid 查询 / 注册用户
   - 生成 JWT token
6. 返回 token 给前端
7. 前端后续请求携带 token

------

## 2. 为什么需要 openid？

- openid 是微信用户在当前小程序中的**唯一标识**
- 用于：用户登录、用户数据绑定

------

## 3. HttpClient 的作用

HttpClient 用于：

- 构造 HTTP 请求（GET/POST）
- 发送请求
- 接收响应

本质：**服务端调用第三方 API（微信服务器）**

------

## 4. JWT 的作用

用于实现 **无状态登录（Stateless Authentication）**

------

## 5. JWT 的组成

三部分：

```
Header.Payload.Signature
```

- Header：算法信息
- Payload：用户数据（userId、过期时间）
- Signature：签名防篡改

------

## 6. 在项目中 JWT 是怎么用的？

- 登录成功后：
  - 生成 JWT token
  - 将 userId、时效写入 payload
- 返回 token 给前端
- 前端请求时：
  - 放在请求头（Authorization / token）

------

## 7. 拦截器做了什么？（重点）

在 Spring 中通过 HandlerInterceptor：

1. 判断是否是需要拦截的请求
2. 获取请求头中的 token
3. 解析 JWT：
   - 成功 → 放行
   - 失败 → 拒绝请求

------

## 8. 为什么用 JWT，而不是 Session？（重点）

| 方案    | 特点                   |
| ------- | ---------------------- |
| Session | 有状态，需要服务器存储 |
| JWT     | 无状态，扩展性好       |

我的选择原因：

- 支持分布式
- 减少服务器压力
- 易扩展

------

## 9. 拓展问题

- JWT 会不会被篡改？
- JWT 如何续期？
- JWT 如何退出登录？
- Token 存在哪更安全？

------

# 三、总结

### 技术点：

- Redis（缓存设计 + 问题排查）
- JWT（认证体系）
- HttpClient（第三方接口调用）
- Spring 拦截器

------

## 学习资料与完整代码

**已整理并上传至 GitHub 仓库**

