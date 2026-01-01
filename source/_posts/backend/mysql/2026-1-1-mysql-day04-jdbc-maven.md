---
title: Maven 与 JDBC 学习笔记：从依赖管理到数据库访问实践
date: 2026-1-1 20:30:00
categories:
  - 后端
  - MySQL
tags:
  - jdbc
  - maven
toc: true
toc_number: true
---

> **一句话导读**
   >  本文系统梳理了 Maven 的核心机制（依赖、生命周期、冲突解决）以及 JDBC 的完整使用流程（连接、CRUD、事务、批处理、连接池），重点理解 Java 程序是如何通过 JDBC 与数据库交互的。

------

   ## 一、Maven

   ### 1. Maven 是什么？

   Maven 是一个 **项目构建与依赖管理工具**。
    当项目需要使用第三方库时，不再手动下载 `.jar` 放进 `lib` 目录，而是通过 Maven **声明依赖 → 自动下载 → 统一管理**。

------

   ### 2. Maven 坐标（Dependency Coordinates）

   每一个依赖在 Maven 中都由一组**唯一坐标**标识：

   - **groupId**：组织或公司名（通常反域名）
   - **artifactId**：项目/模块名
   - **version**：版本号

   示例（EasyExcel）：

   ```
   <dependency>
       <groupId>com.alibaba</groupId>
       <artifactId>easyexcel</artifactId>
       <version>3.1.1</version>
   </dependency>
   ```

   常用坐标查询网站：
	 https://mvnrepository.com/

------

   ### 3. Maven 依赖配置与 scope

   #### 依赖的基本结构

   ```
   <dependencies>
       <dependency>
           <groupId>...</groupId>
           <artifactId>...</artifactId>
           <version>...</version>
           <scope>...</scope>
           <exclusions>...</exclusions>
       </dependency>
   </dependencies>
   ```

   #### 常见依赖范围（scope）

   | scope    | 作用说明                                |
   | -------- | --------------------------------------- |
   | compile  | 默认值，编译 / 测试 / 运行都生效        |
   | test     | 仅测试阶段使用（如 JUnit）              |
   | provided | 编译和测试有效，运行时由容器提供        |
   | runtime  | 编译不需要，运行时需要（**JDBC 驱动**） |
   | system   | 手动指定 jar 路径（不推荐）             |

​    **实践建议**
​    MySQL JDBC 驱动应使用 `runtime`，避免与 Java 标准 JDBC 接口混淆。

------

   ### 4. 依赖传递性与依赖冲突

   #### 4.1 同一 pom 中声明多个版本

   - **后声明的版本生效**

   #### 4.2 传递依赖冲突的解决规则（Maven 依赖调解）

   1. **路径最短优先**
   2. **声明顺序优先**

   如果自动调解不满足需求，需要使用 `<exclusions>` 手动排除依赖。

   - 通常优先保留**较高版本**
   - 若高版本不兼容，考虑升级**上层依赖**

------

   ### 5. Maven 仓库机制

   Maven 仓库类型：

   - **本地仓库**：缓存已下载依赖
   - **远程仓库**
     - 中央仓库
     - 私服
     - 镜像仓库（如阿里云）

   依赖查找顺序：

   1. 本地仓库
   2. 远程仓库
   3. 找不到则报错

------

   ### 6. Maven 生命周期与插件

   #### 生命周期

   - `clean`
   - `default`
   - `site`

   常用阶段：

   - `compile`
   - `install`
   - `deploy`

   ```
   mvn clean install
   ```

   #### 插件机制

   Maven 本质是 **插件执行框架**，如：

   - 编译插件
   - 测试插件
   - Jacoco、Checkstyle、Sonar 等

------

   ### 7. Maven 多模块与最佳实践

   - 父模块：统一管理版本、插件
   - 子模块：只引入实际需要的依赖
   - 使用 `dependencyManagement` 统一版本
   - 将版本号集中在 `<properties>` 中

------

## 二、JDBC

### 1. JDBC 是什么？

JDBC（Java Database Connectivity）是 **Java 访问数据库的标准接口**：

- 接口定义在 Java 标准库中
- 具体实现由数据库厂商提供（JDBC Driver）
- Java 程序通过 JDBC 驱动与数据库通信

------

### 2. JDBC 连接数据库

#### JDBC URL 格式（MySQL）

```
jdbc:mysql://host:port/db?param=value
```

示例：

```
jdbc:mysql://localhost:3306/db01?useSSL=false&characterEncoding=utf8
```

#### 建立连接示例

```
Connection conn = DriverManager.getConnection(url, user, password);
```

 JDBC 连接是昂贵资源，必须及时释放，推荐使用 `try-with-resources`。

------

### 3. JDBC 查询流程（核心）

**标准 4 步：**

1. 加载驱动
2. 获取连接
3. 使用 `PreparedStatement`
4. 使用 `ResultSet` 读取结果

```
Class.forName("com.mysql.cj.jdbc.Driver");

try (Connection conn = DriverManager.getConnection(URL, USER, PASSWORD);
     PreparedStatement ps = conn.prepareStatement("select name, age from tb_user where id <= ?")) {

    ps.setObject(1, 20);
    try (ResultSet rs = ps.executeQuery()) {
        while (rs.next()) {
            System.out.println(rs.getString("name") + " 年龄：" + rs.getInt("age"));
        }
    }
}
```

------

### 4. PreparedStatement 与 SQL 注入

 错误示例（存在 SQL 注入风险）：

```
"SELECT * FROM user WHERE name='" + name + "'"
```

 正确方式：

```
PreparedStatement ps = conn.prepareStatement(
    "SELECT * FROM user WHERE name = ?"
);
ps.setString(1, name);
```

**优势：**

- 防止 SQL 注入
- 可复用 SQL 执行计划
- 性能更好

------

### 5. JDBC 更新操作

- INSERT / UPDATE / DELETE
- 使用 `executeUpdate()`
- 返回影响行数

#### 插入并获取自增主键

```
PreparedStatement ps = conn.prepareStatement(
    sql, Statement.RETURN_GENERATED_KEYS
);

try (ResultSet rs = ps.getGeneratedKeys()) {
    if (rs.next()) {
        long id = rs.getLong(1);
    }
}
```

------

### 6. JDBC 事务

```
conn.setAutoCommit(false);
try {
    // 多条 SQL
    conn.commit();
} catch (SQLException e) {
    conn.rollback();
} finally {
    conn.close();
}
```

我的理解 JDBC 事务的本质：

> **让多条 SQL 在同一个数据库事务中执行**

------

### 7. JDBC 批处理（Batch）

适用于 **同一 SQL，多组参数** 的场景：

```
ps.addBatch();
ps.executeBatch();
```

- 比循环执行效率高
- 返回 `int[]` 表示每条语句影响行数

------

### 8. JDBC 连接池

频繁创建/关闭连接成本高 → 使用连接池复用连接。

#### 常见连接池实现

- HikariCP
- Druid
- C3P0

#### HikariCP 示例

```
HikariConfig config = new HikariConfig();
config.setJdbcUrl("jdbc:mysql://localhost:3306/db01");
config.setUsername("root");
config.setPassword("password");

DataSource ds = new HikariDataSource(config);

try (Connection conn = ds.getConnection()) {
    ...
}
```

 注意：

- `DataSource` 应全局唯一
- `conn.close()` 并不是真正关闭，而是归还连接池

------

## 三、总结与个人理解

通过今天的学习，我对 **Java + 数据库 + Maven** 的理解更加清晰：

1. 体会到了 Maven 的便捷，解决了 **依赖与构建的工程问题**
2. JDBC 是 Java 与数据库之间的 **标准桥梁**（java 核心思想：一次编译到处运行的体现）
3. 事务、批处理、连接池是性能与一致性的关键
4. 而后续要学习的框架（MyBatis / Spring）本质都是在 **JDBC 之上做封装**
