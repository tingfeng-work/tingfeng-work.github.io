---
title: MyBatis XML 映射器学习复盘：结合实践 Bug 
date: 2026-01-04
categories:
  - 后端
  - MyBatis
tags:
  - mybatis
  - xml映射
toc: true
toc_number: true
---

> 在学习 MyBatis XML 映射器之前，我以为这部分内容只是“把 SQL 写进 XML”。
>  但真正动手后，我发现：
>
> **我遇到的大多数问题，都不是 SQL 写错，而是「我并不知道 MyBatis 在什么时候、用什么规则去解析这些 XML」。**
>
> 这篇文章并不是官方文档的复述，而是**我在学习过程中真实遇到的问题、修复的 Bug，以及最终形成理解总结。**

------

## Bug ：为什么必须用 `@Param`，不然 XML 对不上？

###  现象

```
query(String name, String status);
#{name}  // 报错
```

------

### 原因

- 方法传入的形参名不保存，MyBatis 默认参数名为 `param1 / param2`

------

### 正确做法

```
query(@Param("name") String name,
      @Param("status") String status);
```

------

###  最终结论

> **多参数方法，一律使用 `@Param`，及规范又易读**

---------------

## XML 映射器整体结构认知

一个 SQL 映射文件中，真正重要的顶级元素并不多（按推荐顺序）：

- `resultMap`：结果集到对象的映射规则（最强大、最复杂）
- `sql`：可复用的 SQL 片段
- `select`
- `insert`
- `update`
- `delete`

**核心思想只有一个：**

> **Mapper XML 的职责是：定义 SQL + 描述 SQL 执行结果如何映射成 Java 对象。**

------

## select：最常用，也最容易被“想简单”的地方

一个最基本的查询如下：

```
<select id="selectById" resultType="com.tingfeng.mybatis.pojo.User">
  select * from tb_user where id = #{id}
</select>
```

### `#{}` 的本质理解

`#{id}` 并不是字符串拼接，而是 **预编译参数占位符**，底层等价于 JDBC：

```
String sql = "select * from tb_user where id = ?";
PreparedStatement ps = conn.prepareStatement(sql);
ps.setInt(1, id);
```

**结论**：

- `#{}` 安全（防 SQL 注入）
- `#{}` 会使用 `PreparedStatement`

------

### select 的核心属性

| 属性            | 工程级理解                              |
| --------------- | --------------------------------------- |
| `parameterType` | 可省略，MyBatis 可通过参数类型推断      |
| `resultType`    | 适合字段名与属性名一致的简单映射        |
| `resultMap`     | 字段名不一致 / 复杂映射时使用（二选一） |
| `statementType` | 默认 PREPARED，几乎不用改               |
| `resultOrdered` | 针对嵌套结果集的内存优化（进阶）        |

> **resultType 和 resultMap 永远只能选一个**

------

## insert / update / delete：关注事务与主键

### 修改语句的共同点

- insert / update / delete **都会修改数据库**
- **必须提交事务**（`sqlSession.commit()`）

这是我忽略了很多次的一点。

------

### useGeneratedKeys：主键回填

```
<insert id="insertUser"
        useGeneratedKeys="true"
        keyProperty="id">
  insert into tb_user(name, phone)
  values(#{name}, #{phone})
</insert>
```

执行完成后：

```
user.getId(); // 已自动回填
```

> **最常用的主键获取方式**

------

### selectKey：数据库不支持自增时的备用方案

- `BEFORE`：先生成 key，再 insert
- `AFTER`：先 insert，再查询 key（类似 Oracle）

------

## sql 标签：SQL 复用的本质是字符串拼接

```
<sql id="userColumns">
  ${alias}.id, ${alias}.username
</sql>
<select id="selectUsers" resultType="map">
  select
    <include refid="userColumns">
      <property name="alias" value="t1"/>
    </include>
  from tb_user t1
</select>
```

### 关键理解

- `<sql>` 本质是 **字符串模板**
- 所以只能使用 `${}`（字符串替换）
- **不能用于接收用户输入**

**结论**：

- `${}` 只用于 SQL 结构
- `#{}` 只用于业务参数

------

## 参数传递机制：最容易踩坑，踩坑后要通过 sql 日志来分析

### 一个关键前提

> **Mapper 方法的形参名默认不会被保留下来！**

编译后，参数名变成：

```
param1 / param2 / arg0 / arg1
```

------

### 单参数为什么“随便写都能用”

```
User selectUser(int id);
where id = #{id}
```

因为 **只有一个参数，不存在歧义**。

------

### 多参数的真实规则

```
User select(String name, String status);
```

错误写法：

```
where name = #{name} and status = #{status}
```

正确写法（不推荐）：

```
where name = #{param1} and status = #{param2}
```

**推荐写法**（易读，明确）：

```
User select(@Param("name") String name,
            @Param("status") String status);
where name = #{name} and status = #{status}
```

------

### foreach 中 collection 的真相

```
int batchInsert(@Param("users") List<User> users);
<foreach collection="users" item="user">
```

> **collection 指的是映射过程中的“参数名”，不是 Java 变量名**

------

## 结果映射：自动映射 vs resultMap

### 自动映射的前提

- 列名 == 属性名（忽略大小写）
- 或开启下划线转驼峰：

```
<setting name="mapUnderscoreToCamelCase" value="true"/>
```

------

### resultMap 的使用场景

- 列名与属性名完全不一致
- 多表查询（列名冲突）
- 嵌套对象 / 集合映射

```
<resultMap id="userMap" type="User">
  <id column="user_id" property="id"/>
  <result column="user_name" property="username"/>
</resultMap>
```

> `<id>` 与 `<result>` 的区别在于：
>  **id 会被标记为对象标识，对缓存和嵌套映射有优化作用**

------

## 自动映射的工作机制

- 自动映射先执行
- 手动 resultMap 后执行（可覆盖）

自动映射等级：

- `NONE`
- `PARTIAL`（默认）
- `FULL`

------

## 总结

### 动态 SQL / XML 映射的实践原则

1. 参数永远用 `#{}`，结构才用 `${}`
2. 多参数一律使用 `@Param`
3. 字段不一致优先用别名，其次 resultMap
4. insert/update/delete 记得提交事务
5. 通过 **SQL 日志反推问题**，而不是盯 XML 发呆
6. MyBatis 并不“智能”，它只是**严格执行规则**

------

## 写在最后

通过这一阶段对 **XML 映射器** 的系统学习，我最大的收获不是记住了多少标签，而是：

> **真正理解了 MyBatis 是如何把 SQL、参数和 Java 对象串联起来的**

回头看这些 Bug，我发现它们有一个共同点：

> **不是我不会写 SQL，而是我不知道 MyBatis 在“什么时候、用什么身份”去解析这些 XML。**

当我开始用“框架视角”理解 MyBatis，而不是“语法视角”，这些问题才真正消失。



------

## 学习资料与完整代码

- XML 映射器理论学习笔记
- 可运行的 MyBatis Demo

**已整理并上传至 GitHub 仓库（含 README 说明）**



