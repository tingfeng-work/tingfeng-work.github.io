---
title: mysql、mybatis总结复盘
date: 2026-1-6 00:10:00
categories:
  - 杂记
tags:
  - 总结
  - 复盘
  - mysql
  -mybatis
toc: true
toc_number: true
---
# Part A. MySQL（进阶重点 + 我们踩过的坑）

## A1. InnoDB 与索引：必须能讲清的“底层理由”

**重点 1：InnoDB 是索引组织表（IOT）**：数据按主键组织，聚簇索引叶子节点存整行；二级索引叶子节点存主键，回表再找整行。回表是很多慢 SQL 的根。 

**重点 2：B+Tree 为什么是它**：范围查询、排序、IO 友好（高阶、低树高），并且叶子有链表利于区间扫描。 3d511e2d-e4e7-4618-97dd-b861e91…

**重点 3：Explain 怎么用**：别只看 key_len，必须结合 type/rows/extra。

------

## A2. 典型问题（索引/Explain/优化）

### 1）“两条 SQL 的 key_len 都是 54，为啥感觉不一样？”

**现象**：key_len 一样，但一条更慢/走了范围/结果不如预期。
 **原因**：key_len 表示“索引字段使用的字节数”，并不能直接判断“等值还是范围、是否继续利用联合索引后续列”。真正要看的是 **type（ref/range/all）**、rows 估算、extra（Using index / Using where / Using filesort）。3d511e2d-e4e7-4618-97dd-b861e91…
 **面试怎么讲**：我们会说“key_len 是长度指标，不是访问方式指标；访问方式看 type，覆盖/回表看 extra。”

### 2）“外层查询是什么意思？”（典型：limit 深分页优化）

**现象**：看到 `select * from t, (select id from t order by id limit x,y) a where t.id=a.id`。
 **原因**：深分页 `limit offset, size` 会扫描大量行再丢弃；子查询先用覆盖索引拿到一小段 id，再回表取整行，减少扫描与排序成本。
 **面试怎么讲**：我们会强调“先用索引把范围缩小，再回表拿数据”，本质是**用覆盖索引降低代价**。

### 3）“UPDATE 为什么会行锁升级表锁？”

**原因**：InnoDB 行锁是**对索引项加锁**，如果 where 条件没用上索引，会变成扫描并锁很多记录，效果接近表锁（阻塞面急剧扩大）。

 **面试怎么讲**：我们会把关键词说全：**“行锁基于索引项”**、**“索引失效导致锁范围扩大”**

------

## A3. 典型问题（工具/配置/导入/格式）

### 1）`load data local infile` 在 Windows / DataGrip 报错

**常见原因组合**：

- MySQL 服务端 `local_infile` 关闭
- 客户端/驱动禁用了本地导入
- Windows 路径转义、换行符不一致（`\n` vs `\r\n`）
   **解决思路**（实际排查过程）：

1. `show global variables like 'local_infile';` / 必要时 `set global local_infile=1;`
2. DataGrip 连接参数里允许 `allowLoadLocalInfile=true`（不同版本入口不一样，但核心是“驱动放行”）
3. Windows 路径用双反斜杠 `D:\\...\\tb_sku1.sql` 或直接用 `/`；lines terminated 根据文件真实换行设置 `'\r\n'` 或 `'\n'`。
    （这一块属于工程经验点：我们把“权限 + 驱动 + 路径 + 换行”四件套一次排。）

### 2）`Incorrect date value: '2000-01\n 01'`

**原因**：值里混入了换行/空格，MySQL 解析 date 失败。
 **解决**：保证写入字符串严格 `YYYY-MM-DD`，不要跨行；脚本导入时注意编辑器自动换行。
 （这是我们在写 insert 时最典型的“肉眼看不出来”的坑。）

### 3）DataGrip “怎么看慢日志 / 找不到 my.ini”

**要点**：慢日志开关与阈值是 MySQL 变量；配置文件只是“持久化这些变量”的一种方式。我们优先用 SQL 验证：

- `show variables like 'slow_query_log%';`
- `show variables like 'long_query_time';`
   然后再决定是否落到配置文件（Windows 可能是 my.ini，Linux 常见 my.cnf；路径因安装方式不同）。

### 4）GitHub/博客图片“仓库里有但加载不出来”

**高频原因**：相对路径层级写错 / 大小写不一致（GitHub 对路径大小写敏感）/ 引用路径缺少 `./` 或 `../` / 静态站点 baseUrl 影响。
 **习惯**：统一用仓库根目录下 `assets/`，并在 md 里用相对路径（与 md 所在目录一致推导），提交前本地预览一次。

------

## A4. 锁与事务 / MVCC：要能讲到“机制级别”

**重点 1：事务 ACID 实现**：redo/undo + 锁 + MVCC（ReadView + undo 版本链）。
**重点 2：当前读 vs 快照读**：`for update`/`lock in share mode` 是当前读；普通 select 是快照读。
**重点 3：间隙锁/临键锁**：RR 隔离级别下用于防幻读，核心在“锁住索引间隙/范围”。

------

# Part B. MyBatis（从会用到能解释 + 遇过的坑）

## B1. MyBatis 解决了什么问题（对比 JDBC）

我们一开始用 JDBC 会遇到：SQL 硬编码、参数拼接不灵活、结果集解析硬编码、样板代码多；MyBatis 把这些抽成配置与映射，开发关注点回到 SQL 本身。

------

## B2. 架构主线：SqlSession / Executor / MappedStatement

我会用一句话把链路说清：
 **SqlSessionFactory → SqlSession → Mapper 动态代理 → Executor 执行 → MappedStatement 定义入参/出参映射**。

------

## B3. 动态 SQL：必须能写 + 必须能说清“坑点”

我们动态 SQL 的核心组件与易错点如下（都来自我们做过的练习总结）：

### 1）`<choose><when>`：两个条件都传入会怎样？

**结论（重点）**：只命中**第一个满足的 when**，后面的 when 不再判断。

### 2）`<where>`/`<set>`：解决的就是“多余 AND / 结尾逗号 / 空 where”

- `<where>`：自动加 WHERE，并处理开头多余的 AND/OR
- `<set>`：自动处理动态 update 的逗号
   必要时用 `<trim>` 精细控制，这俩标签都是基于 `trim` 的封装

### 3）`<foreach collection="...">`：collection 到底写什么？

**我们问过的关键点**：collection 不是“形参名字一定要叫 users”，而是**MyBatis 对入参的默认命名规则**决定的：

- 单参数 List：默认叫 `list`
- 单参数数组：默认叫 `array`
- 多参数：建议统一用 `@Param("users")` 显式命名，避免“参数名丢失/变成 param1 param2”。
   （这也是我当时纠结“底层不记录形参名”的来源：不加 `-parameters` 或未显式 @Param 时，运行期确实可能拿不到源码形参名。）

### 4）`${}` vs `#{}`：参数注入

**重点**：

- `#{}` 走预编译占位符，安全
- `${}` 直接字符串拼接，有注入风险（比如 order by 字段名、表名拼接最常见）
   **我的处理策略**：配置文件中的 sql 标签元素中可以使用，sql 逻辑条件处不能

------

## B4. 缓存：一二级缓存“存什么、什么时候失效”

- **一级缓存**：SqlSession 级别；同会话相同查询命中；DML/clearCache/回滚等会清。
- **二级缓存**：Mapper namespace 级别，多 SqlSession 可共享；需开启与配置 `<cache>`。
   **面试表达（重点）**：缓存 key 本质由 **statementId + 参数 + 环境信息**组成；value 是查询结果对象（或其序列化形态）。

------

## B5. 我们遇到过的典型报错：ReflectionException “no getter”

**现象**：insert/update 时提示没有 getter。
 **常见原因**：实体类字段名与映射引用不一致 / getter 方法缺失（或 Lombok 未生效）/ XML 写了不存在的属性名。
 **我们的排查路径**：
 1）先看报错指向的 property 名；2）对照实体字段与 getter；3）再看 mapper 中引用是否拼错。

------

# Part C. Maven（把工程习惯固定下来）

**重点 1：依赖坐标与 scope**：JDBC 驱动典型是 runtime；JUnit 是 test；servlet-api 常是 provided。

**重点 2：冲突解决**：最短路径优先 + 声明顺序优先；必要时 exclusions 手动排除。

**重点 3：标准目录结构**：`src/main/java|resources` + `src/test/java|resources`，这个习惯会直接影响后面 Spring Boot 的自动扫描与打包体验。

------

# Part D. 我们把这些复盘怎么变成“可复用的面试回答”

建议把这份复盘拆成 4 组“可背诵问答”（每组 3～5 句即可）：

1. **索引与回表**：为什么慢、怎么建联合索引、什么是覆盖索引
2. **Explain 读法**：type/rows/extra 优先于 key_len
3. **锁与事务**：行锁基于索引项、RR 下 next-key/gap、MVCC 三件套
4. **MyBatis 动态 SQL 与安全**：choose/where/set/foreach、@Param、#{}/ ${}
