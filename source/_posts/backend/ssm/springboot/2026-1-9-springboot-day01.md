---
title: Spring Boot 基础入门与整合实践
date: 2026-01-09
categories:
  - 后端
  - SpringBoot
tags:
  - mybatis
  - druid
toc: true
toc_number: true
---

> 本文记录了我在学习 Spring Boot 基础与整合 MyBatis、Druid 过程中的核心知识点，以及一次完整、真实的排错过程。
>重点不在于“配置怎么写”，而在于**理解 Spring Boot 自动配置的工作机制**。

------

## 一、Spring Boot 简介

Spring Boot 是由 Pivotal 团队提供的快速开发框架，目标是：

- 简化 Spring 应用的初始搭建
- 减少繁琐的 XML / Java 配置
- 通过 **自动配置（Auto Configuration）+ Starter 机制** 提升开发效率

------

## 二、Spring Boot 入门方式

### 2.1 创建方式说明

- 使用 IDEA / 官网 / 阿里云脚手架：**需要联网**
- 手动创建 Maven 项目：
  - 继承 `spring-boot-starter-parent`
  - 手动编写启动类
  - **不需要联网**（前提是本地仓库已有依赖）

------

## 三、`parent` 与 `starter` 的作用区分

### 3.1 parent：统一依赖版本管理

`spring-boot-starter-parent` 的作用是：

- 统一管理常用依赖的版本
- 避免版本冲突
- 简化 pom 文件

> **注意**：
>  parent 只“管理版本”，并不会实际引入任何依赖。

------

### 3.2 starter：功能级依赖集合

starter 的作用是：

- 一次性引入某个技术栈所需的依赖
- 通过**依赖传递**减少配置成本

例如：

- `spring-boot-starter-web`
- `spring-boot-starter-jdbc`
- `mybatis-spring-boot-starter`

> parent + starter 解决的是 **“配置复杂度”问题**

------

## 四、Spring Boot 启动类

```
@SpringBootApplication
public class Springboot01QuickstartApplication {
    public static void main(String[] args) {
        SpringApplication.run(Springboot01QuickstartApplication.class, args);
    }
}
```

### 启动类的作用

- 标识这是一个 Spring Boot 应用
- 启动 Spring 容器
- 扫描当前包及其子包下的 Bean
- 返回值是 `ApplicationContext`

------

## 五、内嵌 Web 容器机制

在引入 `spring-boot-starter-web` 后：

- 默认内嵌 Tomcat
- Spring 会创建并管理一个 Tomcat 对象
- 启动应用时自动启动 Web 服务器

Spring Boot 也支持 Jetty、Undertow 等内嵌容器。

------

## 六、基础配置文件

### 6.1 配置文件类型与优先级

Spring Boot 支持：

- `application.properties`
- `application.yml`
- `application.yaml`

优先级（从高到低）：

```
properties > yml > yaml
```

相同配置高优先级覆盖，未冲突配置全部生效。

------

### 6.2 为什么推荐 YAML

- 层级清晰
- 可读性更好
- 更适合复杂配置

#### YAML 核心规则

- 大小写敏感
- 缩进表示层级（只能用空格）
- `key: value` 中冒号后必须有空格
- `#` 表示注释

------

### 6.3 YAML 数据绑定方式

#### 方式一：`@Value`

```
@Value("${enterprise.name}")
private String name;
```

#### 方式二：`Environment`

```
environment.getProperty("enterprise.name");
```

#### 方式三（推荐）：`@ConfigurationProperties`

```
@Component
@ConfigurationProperties(prefix = "enterprise")
@Data
public class Enterprise {
    private String name;
    private int age;
    private String tel;
    private String[] hobby;
}
```

> 本质：
>  Spring Boot 内部大量使用这种方式完成第三方组件的**自动配置**

------

## 七、整合第三方技术

### 7.1 Spring Boot 整合 JUnit

- 自动引入测试 starter
- 使用 `@SpringBootTest`
- 测试类不在启动类包下时需指定 `classes`

------

### 7.2 Spring Boot 整合 MyBatis

MyBatis 主要涉及两类配置：

1. **全局配置**（数据源）
2. **映射配置**（XML / 注解）

基本步骤：

- 引入 MyBatis Starter
- 配置数据源
- 定义 Mapper 接口
- 注入 Mapper 测试

------

### 7.3 Spring Boot 整合 Druid

- 引入 Druid Starter
- 配置 Druid 专属配置项
- 验证 DataSource 类型

------

## 八、整合过程中的问题与排错总结（重点）

### 问题一：`UserDao` 无法注入

#### 现象

```
No qualifying bean of type 'tingfeng.mapper.UserDao' available
```

#### 原因

- Mapper 接口未被 Spring 扫描
- `@Mapper` 只对 MyBatis 生效

#### 解决方式

```
@SpringBootApplication
@MapperScan("tingfeng.mapper")
public class Quickstart03Application { }
```

#### 关键认知

> **@Mapper 是 MyBatis 的，@MapperScan 才是 Spring 的**

------

### 问题二：Mapper 能扫描，但报 `sqlSessionFactory` 缺失

#### 现象

```
Property 'sqlSessionFactory' or 'sqlSessionTemplate' are required
```

#### 定位结论

- Mapper 已创建
- SqlSessionFactory 未生成
- SqlSessionFactory 依赖 DataSource

#### 实际解决方式

> **将 `mybatis-spring-boot-starter` 从 3.0.4 升级至 4.0.2**

#### 根因分析（重要）

新版本调整了：

- Mapper 初始化时机
- SqlSessionFactory 校验逻辑

避免在 DataSource 尚未完全初始化时进行强校验，从而绕开了该异常路径。

> ⚠️ 注意：
>  这并不代表 DataSource 问题“消失”，而是**校验时机发生了变化**

------

## 九、总结与反思

这次学习和排错过程中，我最大的收获不是“配置记住了多少”，而是：

- 理解了 **Starter 才是自动配置的触发条件**
- 明白了 **异常信息可能并不是根因**
- 学会从 **DataSource → SqlSessionFactory → Mapper** 反向排查
- 认识到 **版本升级可能改变行为，而非修复根因**

------

## 十、一句话总结

> Spring Boot 整合 MyBatis 与数据源时，
>  问题往往不在 Mapper 本身，
>  而在自动配置链路是否完整、以及校验时机是否合理。

------

## 学习资料与完整代码

**已整理并上传至 GitHub 仓库**

