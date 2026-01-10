---
title: Spring Boot + SSMP 基础实战与问题复盘
date: 2026-01-010
categories:
  - 后端
  - SpringBoot
tags:
  - mybatis-plus
  - Spring MVC
  - Spring
toc: true
toc_number: true
---

> 本文基于一次完整的 Spring Boot 基础项目实践，总结了从**实体类 → 数据层 → 业务层 → 表现层 → 前后端联调**的开发流程，并重点记录了在整合 MyBatis-Plus、JUnit、分页插件等过程中遇到的典型问题及解决方案，作为后续复盘与查错参考。

项目采用 **单体架构（非前后端分离）**，以熟悉 Spring Boot + MyBatis-Plus 的基础开发模式为目标。

## 一、实体类开发（Entity）

### 1. 数据库准备

- 创建业务表（如 `tbl_book`）
- 初始化测试数据

### 2. 实体类创建

- 根据表结构创建实体类
- 使用 **Lombok** 简化样板代码（getter / setter / toString 等）

```
@Data
public class Book {
    private Long id;
    private String type;
    private String name;
    private String description;
}
```

------

## 二、数据层开发（CRUD）

### 技术选型

- ORM 框架：**MyBatis-Plus**
- 数据源：**Druid**
- 测试框架：JUnit

### 核心步骤

1. 引入 MyBatis-Plus Starter
2. 配置数据库连接信息
3. 配置 MP 相关属性
   - 表名前缀（`table-prefix`）
   - 主键策略（`id-type`）
   - SQL 日志（`log-impl`）
4. 使用 `BaseMapper<T>` 快速完成 CRUD
5. 使用 `@Mapper` 或 `@MapperScan` 交给 Spring 管理
6. 编写 Mapper 测试类验证功能

------

## 三、问题复盘 ①：测试类能运行，测试方法却报 `NoSuchMethodError`

### 现象

- 直接运行测试类 ✔
- 单独运行测试方法 ❌ 报错：`NoSuchMethodError`

### 原因分析

- **Classpath 中存在多个 JUnit 版本**
- Spring Boot 默认引入的测试依赖与本地环境冲突

### 解决过程

- 尝试在 `pom.xml` 中强制覆盖 JUnit 版本 → ❌ 无效
- **最终解决方案：降低 Spring Boot 版本**

从 **Spring Boot 4.x 降级到 Spring Boot 3.x** 后问题消失

> 结论：
>  **测试相关问题优先排查：Spring Boot 版本 × Starter 版本 × IDE 运行方式**

------

## 四、问题复盘 ②：Spring Boot 3 + MyBatis-Plus 启动时报 Mapper Bean 异常

### 报错信息

```
Invalid bean definition with name 'xxxMapper'
```

### 排查过程

- `@Mapper` ✔
- `@MapperScan` ✔
- 包路径无误 ✔

### 根本原因

**Spring Boot 3 与 MyBatis-Plus Starter 版本不兼容**

### 正确依赖方式

```
<!-- Spring Boot 2.x -->
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-spring-starter</artifactId>
</dependency>

<!-- Spring Boot 3.x -->
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-spring-boot3-starter</artifactId>
</dependency>
```

> 结论：
>  **Spring Boot 3 必须使用 boot3 专用的 MyBatis-Plus Starter**

------

## 五、问题复盘 ③：配置了 `IdType.AUTO`，却生成了“雪花 ID”

### 现象

- 实体类主键配置为 `IdType.AUTO`
- 插入后 ID 却像雪花算法生成

### 真正原因

并非 AUTO 失效，而是：

- 之前使用过 **雪花 ID 策略**
- 删除数据后，**数据库的 `AUTO_INCREMENT` 未重置**
- 产生了“脏 ID ”

### 解决方式

```
ALTER TABLE tbl_book AUTO_INCREMENT = 1;
```

> 结论：
>  **主键策略问题，一定要同时检查：代码配置 + 数据库状态**

------

## 六、数据层开发（分页功能）

### MyBatis-Plus 分页 API

```
@Test
void testGetPage(){
    IPage<Book> page = new Page<>(2, 5);
    bookMapper.selectPage(page, null);

    System.out.println(page.getCurrent());
    System.out.println(page.getSize());
    System.out.println(page.getTotal());
    System.out.println(page.getPages());
    System.out.println(page.getRecords());
}
```

### 分页参数说明

- 当前页码
- 每页条数

### 必须配置分页拦截器（由于分页是方言，为了提高扩展性，通过拦截器的方式实现）

```
@Configuration
public class MPConfig {

    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor(){
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        interceptor.addInnerInterceptor(new PaginationInnerInterceptor());
        return interceptor;
    }
}
```

------

## 七、问题复盘 ④：分页不生效

### 原因

- MyBatis-Plus **将分页能力拆分为插件**
- 未引入解析 SQL 的依赖

### 解决方案

```
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-jsqlparser</artifactId>
    <version>3.5.15</version>
</dependency>
```

------

## 八、数据层开发（条件查询）

### 普通条件构造（存在风险）

```
QueryWrapper<Book> qw = new QueryWrapper<>();
qw.like("name", "Spring");
bookMapper.selectList(qw);
```

❌ 问题：字段名是字符串，**编译期无法检查**

------

### Lambda 条件构造（推荐）

```
LambdaQueryWrapper<Book> lqw = new LambdaQueryWrapper<>();
lqw.like(name != null, Book::getName, name);
bookMapper.selectList(lqw);
```

✅ 优点：

- 编译期安全
- 支持条件开关
- 重构友好

------

## 九、业务层开发（Service）

业务层职责：

> **组织业务逻辑，对数据层进行封装调用**

开发步骤：

1. 定义 Service 接口
2. 编写 ServiceImpl
3. 注入 Mapper
4. 编写测试验证逻辑正确性

------

## 十、表现层开发（Controller）

### 核心工作

- 编写 REST Controller
- 接收参数
- 调用业务层
- 返回统一结果

### 参数接收注意点

- JSON 实体：`@RequestBody`
- 路径变量：`@PathVariable`

使用 **ApiFox** 进行接口测试与调试。

------

## 十一、统一返回结果设计

为保证前后端交互一致性：

- 封装统一响应对象
- 包含：
  - `code`
  - `data`
  - `msg`
- 同时考虑异常场景

------

## 十二、前后端联调

- 将前端静态资源拷贝到：

  ```
  resources/static
  ```

- Spring Boot 自动托管静态资源

- 实现简单的前后端一体化访问

------

## 总结

通过这次 Spring Boot 基础实战，我完成了：

- 一条完整的 **SSMP 开发链路**
- 多个 **真实问题的排查与解决**
- 对 **版本兼容性、插件机制、主键策略** 的深入理解

## 学习资料与完整代码

**已整理并上传至 GitHub 仓库**

