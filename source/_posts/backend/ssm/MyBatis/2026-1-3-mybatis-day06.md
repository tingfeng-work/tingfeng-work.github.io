---
title: MyBatis 动态 SQL 五个 Level 实战复盘：从语法理解到踩坑总结
date: 2026-01-03
categories:
  - 后端
  - MyBatis
  - ssm
tags:
  - mybatis
  - 动态mysql
toc: true
toc_number: true
---

> 在学习 MyBatis 动态 SQL 的过程中，我没有只停留在“看文档 + 记语法”，而是通过 **5 个循序渐进的练习 Level**，把 `if / where / trim / choose / foreach` 等标签真正用到了代码中。
>
> 这篇文章不是对官方文档的复述，而是对 **实际编写动态 SQL 时遇到的问题、报错和解决方案的系统复盘**。

------

## 一、为什么要写这篇文章？

MyBatis 动态 SQL 看起来很简单：

```
<if>、<where>、<trim>、<foreach>
```

但真正写起来，会频繁遇到：

- SQL 语法错误
- 查不到数据但不报错
- 参数传了却没生效
- `AND / WHERE / ,` 到底谁该写谁不该写
- `<foreach>` 里 `collection` 到底写什么

**这些问题，光看文档是很难意识到的，必须通过实战踩坑。**

------

## 二、动态 SQL 常用标签速览（不展开）

本文默认你已经了解以下标签的基本用法：

- `<if>`
- `<where>`
- `<trim>`
- `<choose / when / otherwise>`
- `<foreach>`
- `<set>`

**完整的理论学习笔记 + 示例代码我已整理在 GitHub 仓库中**，本文重点放在「实践中遇到的问题」。

------

## Exercise 1：`<if>` + `WHERE` —— SQL 语法错误的第一个坑

### 问题现象

执行查询时报错：

```
You have an error in your SQL syntax
```

日志中看到实际 SQL：

```
select * from tb_user
WHERE name = ?, 
      and gender = ?, 
      and status = ?
```

### 错误原因

- 把 `,` 当成条件连接符（这是 INSERT / UPDATE 才用的）
- 手写 `WHERE`，又在 `<if>` 中手写 `AND`
- 导致 SQL 结构非法

### 正确写法

```
<select id="queryUser" resultType="User">
  select * from tb_user
  <where>
    <if test="name != null and name != ''">
      name = #{name}
    </if>
    <if test="gender != null">
      and gender = #{gender}
    </if>
    <if test="status != null">
      and status = #{status}
    </if>
  </where>
</select>
```

### 经验总结

> **条件查询不要手写 WHERE，让 `<where>` 帮你管理 AND**
>
> 可以先写一条正常的 sql，那这道题举例：
>
> ​	select * from tb_user where name = ? and gender = ? and status = ?;
>
> 这样对照这条 sql 去写动态 sql 会比较不容易出错一点

------

## Exercise 2：`<where>` vs `<trim>` —— 前缀修剪机制

###  疑问

```
<trim prefix="WHERE" prefixOverrides="AND |OR ">
```

- `prefixOverrides` 是干什么的？
- 为什么后面有空格？

### 正确理解

MyBatis 是**字符串拼接 SQL**：

```
<trim prefix="WHERE" prefixOverrides="AND |OR ">
  <if test="name != null">
    AND name = #{name}
  </if>
</trim>
```

执行逻辑是：

1. 先拼出：`AND name = ?`
2. 删除前缀 `AND `
3. 再加上 `WHERE`

### 经验总结

> **`<where>` 本质是 MyBatis 封装好的 `<trim>`**

------

## Exercise 3：数据库 `char(1)` × Java `char` —— 查不到数据但不报错

### 问题现象

- SQL 正常执行
- `Total: 0`
- 数据库中明明存在数据却查不到

日志中曾出现：

```
and gender = '\0'
```

### 根因分析（关键坑）

数据库字段：

```
gender char(1)
```

Java 实体类写成了：

```
private char gender;
```

但：

- Java `char` 默认值是 `'\0'`
- JDBC 会绑定成空字符
- 数据库中不存在这种值

### 正确映射方式

> **数据库 char / varchar → Java 用 String，不要用 char**

```
private String gender;
private String status;
user.setGender("1");
user.setStatus("0");
```

### 经验总结（非常重要）

> **数据库 char ≠ Java char**
>  Java `char` 是 Unicode 字符，不是字符串

------

## Exercise 4：`<choose>` —— 只会执行一个分支

### 疑问

> 如果多个条件同时满足，会不会都执行？

### 实际行为

```
<choose>
  <when test="title != null">
    AND title like #{title}
  </when>
  <when test="author != null">
    AND author = #{author}
  </when>
  <otherwise>
    AND featured = 1
  </otherwise>
</choose>
```

- **只执行第一个满足条件的 `when`**
- 后面的直接忽略

### 经验总结

> **`<choose>` = SQL 版 switch-case**

------

## Exercise 5：`<foreach>` —— collection 到底写什么？

### 疑问

> `collection` 不是 Java 形参名吗？

### ❌ 易错点

```
int batchInsert(List<User> users);
```

如果不加 `@Param`：

- XML 中 **不能写 `users`**

- MyBatis 默认参数名是 `list`

> MyBatis 运行时默认拿不到方法形参名（比如 `users`），所以它只能用自己生成的默认名字（`list` / `collection` / `param1`…）
>
> 因为 Java 编译后的 `.class` 文件里，**默认不会保留形参名**（会变成 `arg0/arg1`）
>
> 加了 @Param 后，因为 `@Param` 是你主动给 MyBatis 一个**稳定的参数名**，它不需要依赖反射去猜

### 推荐写法（最清晰）

```
int batchInsert(@Param("users") List<User> users);
<foreach collection="users" item="user" separator=",">
  (#{user.name}, #{user.phone})
</foreach>
```

### 经验总结

> **collection = 集合参数名**
>  **item = 集合中的单个元素**

------

## 六、五个 Level 练习后的工程级总结

### 动态 SQL 编写原则

1. 条件查询永远用 `<where>` / `<trim>`
2. 更新语句永远用 `<set>`
3. 条件连接符只有 and/or，没有 `,`
4. 数据库 char / varchar → Java 用 String
5. `<choose>` 只会命中一个分支
6. `<foreach>` 建议配合 `@Param`

------

## 七、学习资料与完整代码

- 动态 SQL 理论学习笔记
- 五个 Level 的完整练习代码
- 可运行的 MyBatis Demo

👉 **已整理并上传至 GitHub 仓库（含 README 说明）**

------

## 八、写在最后

通过这五个 Level 的练习，我最大的收获不是“记住了多少标签”，而是：

> **学会了如何通过 SQL 日志反推 XML 和 Java 代码的问题**，然后根据问题动态完善自己的 sql 语句编写

这也是我认为学习 MyBatis 动态 SQL **最重要的能力**。

