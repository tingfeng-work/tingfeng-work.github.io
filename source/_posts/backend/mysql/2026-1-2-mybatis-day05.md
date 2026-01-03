---
title: MyBatis 入门学习笔记：从 JDBC 到 Mapper 映射
date: 2026-01-02 
categories:
  - 后端
  - MySQL
tags:
  - jdbc
  - mybatis
  - mysql
toc: true
toc_number: true
---

> **一句话导读**
   >   MyBatis 的核心价值不在于“帮你写 SQL”，而在于 **消除 JDBC 中大量的硬编码问题，让 SQL 与 Java 解耦，同时保持对 SQL 的完全掌控**。

------

## 一、为什么需要 MyBatis？—— 从 JDBC 的痛点说起

在直接使用 JDBC 操作数据库时，很快会遇到以下问题：

### SQL 硬编码问题严重

- SQL 直接写在 Java 代码中
- 一旦 SQL 发生变化，必须修改并重新编译 Java 代码
- 实际项目中，SQL 变化非常频繁，维护成本高

------

### 参数绑定不灵活

- 使用 `PreparedStatement` 需要手动设置占位符
- `where` 条件的数量和顺序变化时：
  - SQL 要改
  - Java 代码也要跟着改
- 可维护性较差

------

### 结果集解析硬编码

- 查询结果需要手动从 `ResultSet` 中取值
- 强依赖数据库字段名
- 一旦 SQL 查询字段变化，解析逻辑也要修改

------

### 总结一句话

> **JDBC 的问题本质是：硬编码太多、关注点不分离、维护成本高。**

因此，引入 **MyBatis** 来解决这些问题。

------

## 二、MyBatis 是什么？

**MyBatis 是一个持久层（DAO 层）框架**，对 JDBC 操作过程进行了封装，使开发者：

- 只需要关注 **SQL 本身**
- 不再关心：
  - 驱动注册
  - Connection 创建
  - Statement / PreparedStatement 创建
  - 参数设置
  - 结果集解析

------

### MyBatis 的核心思想

- 使用 **XML 或注解** 描述 SQL
- 将 SQL 与 Java 对象进行映射
- 框架负责：
  - 执行 SQL
  - 参数映射
  - 结果集映射
  - 返回 Java 对象

------

## 三、MyBatis 整体架构与工作流程

###  核心配置文件与映射文件

- **`mybatis-config.xml`**

  全局配置文件，配置：

  - 数据源
  - 事务管理器
  - Mapper 映射文件位置

- **`mapper.xml`**

  - SQL 映射文件
  - 一个 SQL 对应一个 `MappedStatement`

------

### SqlSessionFactory 与 SqlSession

- `SqlSessionFactoryBuilder`
   → 构建 `SqlSessionFactory`
- `SqlSessionFactory`
   → 创建 `SqlSession`
- **所有数据库操作都通过 `SqlSession` 完成**

------

### Executor（执行器）

MyBatis 底层通过 `Executor` 接口执行 SQL：

- **SimpleExecutor**：基本执行器
- **CachingExecutor**：带缓存的执行器

------

### MappedStatement（核心抽象）

- 每一条 SQL 都会被封装成一个 `MappedStatement`
- 包含：
  - SQL 本身
  - 输入参数映射规则
  - 输出结果映射规则

 本质类比 JDBC：

- 参数映射 ≈ `PreparedStatement.setXxx`
- 结果映射 ≈ `ResultSet` 解析

------

### MyBatis 工作流程总结

1. 构建 `SqlSessionFactory`
2. 获取 `SqlSession`
3. 获取 Mapper 接口代理对象
4. 调用接口方法执行 SQL
5. 提交事务并关闭 Session

------

## 四、MyBatis 的核心特性

- **SQL 与 Java 解耦**
- **自动结果映射**
- **强大的动态 SQL 能力**
- **内置缓存机制（一级 / 二级）**

------

## 五、MyBatis 缓存机制

###  一级缓存（SqlSession 级别）

- 默认开启、不可关闭
- 同一个 `SqlSession` 中，相同查询只执行一次 SQL

**触发清空的情况：**

- 执行 `INSERT / UPDATE / DELETE`
- 调用 `clearCache()`
- 事务回滚
- 不同的 `StatementId`

------

### 二级缓存（Mapper 级别）

- 多个 `SqlSession` 共享
- 需要显式开启

#### 全局配置

```
<settings>
    <setting name="cacheEnabled" value="true"/>
</settings>
```

#### Mapper 配置

```
<cache
  eviction="LRU"
  flushInterval="60000"
  size="512"
  readOnly="true"/>
```

#### 常见缓存策略

| 策略 | 说明                 |
| ---- | -------------------- |
| LRU  | 最近最少使用（默认） |
| FIFO | 先进先出             |
| SOFT | 软引用               |
| WEAK | 弱引用               |

------

## 六、MyBatis 入门实践

### 引入依赖

```
<dependency>
  <groupId>org.mybatis</groupId>
  <artifactId>mybatis</artifactId>
  <version>x.x.x</version>
</dependency>
```

------

### 构建 SqlSessionFactory（XML）

```
InputStream is = Resources.getResourceAsStream("mybatis-config.xml");
SqlSessionFactory factory = new SqlSessionFactoryBuilder().build(is);
```

------

### Mapper XML 示例

```
<mapper namespace="UserMapper">
    <select id="getUser" resultType="User">
        SELECT * FROM tb_user WHERE id = #{id}
    </select>
</mapper>
```

------

### 使用 SqlSession 执行 SQL（旧方式）

```
List<User> list = session.selectList("UserMapper.getUser", 10);
```

问题：

- namespace, select ID 硬编码
- 参数类型不安全

------

### Mapper 接口（推荐方式）

```
public interface UserMapper {
    List<User> getUser(int id);
}

UserMapper mapper = session.getMapper(UserMapper.class);
mapper.getUser(10);
```

**底层通过动态代理生成实现类**，接口方法 ↔ SQL 映射一一对应。

------

### Mapper 接口与 XML 的目录约定（重要）

- Mapper 接口在 `java` 目录
- Mapper XML 在 `resources` 目录
- **包路径保持一致**

这是 MyBatis 正确定位 SQL 的关键。

------

## 七、注解开发方式

### 基本示例

```
@Select("SELECT * FROM tb_user WHERE id = #{id}")
User getUser(int id);
```

### 常用注解

- `@Select / @Insert / @Update / @Delete`
- `@Param`：多参数映射
- `@Results / @Result`：字段映射

```
@Results({
    @Result(property = "id", column = "user_id"),
    @Result(property = "name", column = "user_name")
})
```

**官方建议**：

- 简单 SQL → 注解
- 复杂 SQL → XML

------

## 八、工具类封装 SqlSessionFactory

由于获取连接流程固定，可以封装工具类：

```
InputStream is = Resources.getResourceAsStream("mybatis-config.xml");
SqlSessionFactory factory = new SqlSessionFactoryBuilder().build(is);
SqlSession session = factory.openSession();
```

------

## 九、总结与个人理解

通过 MyBatis 的学习，我对 Java 数据访问层有了更清晰的认识：

1. MyBatis 的核心价值是 **解耦，而不是隐藏 SQL**
2. 它通过配置与映射，消除了 JDBC 的大量硬编码
3. Mapper 接口 + 动态代理，是非常优雅的设计
4. XML 与注解并不是对立关系，而是各司其职

