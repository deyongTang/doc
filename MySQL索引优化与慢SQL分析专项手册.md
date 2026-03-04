# MySQL 索引优化与慢 SQL 分析专项手册

> 适合 Java 工程师 · 从原理到实战 · 可落地可复用

---

## 目录

1. [MySQL 索引基础与底层原理](#第一章mysql-索引基础与底层原理)
2. [索引类型详解与使用场景](#第二章索引类型详解与使用场景)
3. [索引优化核心规则与失效场景](#第三章索引优化核心规则与失效场景)
4. [EXPLAIN 执行计划深度解读](#第四章explain-执行计划深度解读)
5. [慢 SQL 定位与排查工作流](#第五章慢-sql-定位与排查工作流)
6. [常见优化案例与 SQL 改写技巧](#第六章常见优化案例与-sql-改写技巧)
7. [其他 MySQL 性能优化手段](#第七章其他-mysql-性能优化手段)
8. [生产环境排查 SOP](#第八章生产环境排查-sop)

---

## 第一章：MySQL 索引基础与底层原理

### 1.1 为什么要有索引？

用 Java 类比来理解：

> 💡 **类比**：全表扫描 = 遍历 `ArrayList`（O(n)）；索引 = `HashMap` 桶定位（O(1)）或 `TreeMap` 二叉查找（O(log n)）。索引的本质就是**用空间换时间**。

没有索引时，MySQL 查找一条记录需要扫描整张表的每一行；有了索引，MySQL 能快速定位到目标数据的磁盘位置。

---

### 1.2 B+ 树：InnoDB 的核心数据结构

InnoDB 的索引底层使用 **B+ 树（B+Tree）**，理解 B+ 树是理解所有索引行为的基础。

#### B+ 树核心特征

- 所有数据都存储在**叶子节点**，非叶子节点只存索引键
- 叶子节点之间通过**双向链表**连接（支持高效范围查询）
- 树的高度通常只有 **2~4 层**（百万级数据只需 3 次 IO）
- 每个节点对应磁盘上的一个**数据页**（默认 16KB）

> 💡 **Java 类比**：B+ 树就像一个多层的 `TreeMap`，上层是路由索引，最底层才是真实数据，且底层所有节点都串成了一个**有序链表**。

| 特性 | B 树 | B+ 树（InnoDB 使用） |
|------|------|---------------------|
| 数据存储位置 | 所有节点都存数据 | 只有叶子节点存数据 |
| 叶子节点 | 不相互连接 | 双向链表连接 |
| 范围查询 | 需要中序遍历，效率低 | 直接遍历叶子链表，高效 |
| 磁盘 IO | 相对较多 | 相对较少 |
| 场景适用 | 文件系统 | 数据库索引（MySQL InnoDB） |

---

### 1.3 聚簇索引 vs 非聚簇索引（二级索引）

这是 InnoDB 最重要的概念之一，很多行为都源于此。

#### 聚簇索引（Clustered Index）

- 每张 InnoDB 表**有且只有一个**聚簇索引
- 叶子节点存储的是**完整的行数据**（不是指针）
- 默认使用主键作为聚簇索引；没有主键则用第一个唯一非空索引；都没有则内部生成 rowid

> ⚠️ **注意**：主键建议使用**自增整数**，避免使用 UUID（随机写导致频繁页分裂，性能差）。

#### 非聚簇索引（Secondary Index / 二级索引）

- 叶子节点存储：**索引列的值 + 主键值**（不是行数据本身）
- 查询时先找到主键值，再回到聚簇索引查完整数据，这个过程叫「**回表**」
- 回表有额外 IO 开销，优化目标之一就是**减少回表**

> 💡 **Java 类比**：聚簇索引 = `HashMap<主键, 整个对象>`；非聚簇索引 = 另一个 `HashMap<索引列, 主键>`，查到主键后再去第一个 HashMap 拿完整对象（这个二次查找就是回表）。

| 对比维度 | 聚簇索引 | 非聚簇索引（二级索引） |
|---------|---------|----------------------|
| 叶子节点内容 | 完整行数据 | 索引列 + 主键值 |
| 数量限制 | 每表只有一个 | 可以有多个 |
| 查询速度 | 直接获得数据 | 需要回表（额外一次查找） |
| 写入影响 | 主键插入顺序影响性能大 | 每个二级索引都增加写入开销 |

---

### 1.4 覆盖索引：避免回表的利器

**核心定义**：如果一个查询所需的所有列都包含在某个索引的叶子节点中（索引列 + 主键列），则该查询不需要回表，这种情况称为「**覆盖索引**」。

EXPLAIN 中 `Extra` 列显示 `Using index` 即表示用到了覆盖索引。

```sql
-- 表: user(id PK, name, age, email)
-- 有索引: idx_name_age(name, age)

-- ✅ 覆盖索引：查询列 name、age、id 都在索引叶子节点里，无需回表
SELECT id, name, age FROM user WHERE name = '张三';

-- ❌ 需要回表：email 不在索引里，需要回聚簇索引取完整行
SELECT id, name, age, email FROM user WHERE name = '张三';
```

> ✅ **最佳实践**：对于高频查询，可以通过把查询列加入联合索引来实现覆盖索引，消除回表开销。

---

## 第二章：索引类型详解与使用场景

### 2.1 主键索引（Primary Key Index）

- 就是聚簇索引，B+ 树叶子存完整行数据
- 自动唯一且非空
- 强烈建议使用自增 `INT/BIGINT`，**禁止使用 UUID 作主键**

### 2.2 普通索引（Normal Index）

```sql
CREATE INDEX idx_name ON user(name);
```

- 最基础的二级索引，无唯一约束
- 叶子节点存储 name 列值 + 主键 id

### 2.3 唯一索引（Unique Index）

```sql
CREATE UNIQUE INDEX uk_email ON user(email);
```

- 比普通索引多了唯一约束检查
- 查找时找到第一条匹配记录就停止

> 💡 **提示**：唯一索引在 InnoDB 中不能使用 Change Buffer（插入时必须立即读磁盘校验唯一性），写入性能略低于普通索引。如果业务层已保证唯一，用普通索引性能更好。

### 2.4 联合索引（Composite Index）

```sql
CREATE INDEX idx_name_age_city ON user(name, age, city);
```

联合索引是日常优化中最重要的索引类型，核心规则是「**最左前缀匹配**」。

#### 最左前缀匹配规则

联合索引 `(name, age, city)` 实际上包含了以下几个索引的能力：
- `(name)`
- `(name, age)`
- `(name, age, city)`

但**不包含** `(age)`、`(city)`、`(age, city)` 的能力。

| 查询条件 | 是否用到索引 | 命中哪些列 | 说明 |
|---------|------------|----------|------|
| `WHERE name = ?` | ✅ 是 | name | 命中最左列 |
| `WHERE name = ? AND age = ?` | ✅ 是 | name, age | 命中前两列 |
| `WHERE name = ? AND age = ? AND city = ?` | ✅ 是 | name, age, city | 命中全部 |
| `WHERE age = ?` | ❌ 否 | 无 | 跳过了 name |
| `WHERE name = ? AND city = ?` | ⚠️ 部分 | name | age 断了，city 无法命中 |
| `WHERE name LIKE '张%' AND age = ?` | ⚠️ 部分 | name | LIKE 前缀可用，但 age 无法继续 |

> ⚠️ **注意**：联合索引中，一旦某列使用了范围查询（`>`、`<`、`BETWEEN`、`LIKE` 前缀），后面的列就无法再走索引。

### 2.5 前缀索引

```sql
CREATE INDEX idx_email_prefix ON user(email(20));
```

- 只对字符串列的前 N 个字符建索引，节省空间
- 缺点：**无法使用覆盖索引**（存的是截断值，不能还原完整值）
- 适用于长字符串列（如 URL、email）

> 💡 **前缀长度选取**：`SELECT COUNT(DISTINCT LEFT(email, N)) / COUNT(*) FROM user`，找到接近 1 的最小 N 值。

### 2.6 索引设计原则总结

| 原则 | 说明 |
|------|------|
| 选择性高的列优先 | 区分度越高，索引越有效（性别列不适合建索引） |
| 频繁出现在 WHERE/JOIN/ORDER BY 的列 | 这些是索引的主要受益场景 |
| 联合索引列顺序：等值在前，范围在后 | 等值查询的列放左边，范围查询的列放右边 |
| 覆盖索引优先 | 查询列能被索引覆盖，避免回表 |
| 索引不是越多越好 | 每个索引增加写入开销，一张表索引控制在 5 个以内 |
| 避免冗余索引 | `(name)` 和 `(name, age)` 共存时，`(name)` 是冗余的 |

---

## 第三章：索引优化核心规则与失效场景

> 这是面试高频考点，也是日常踩坑的重灾区，必须熟记。

### 场景一：对索引列做函数运算或类型转换

```sql
-- ❌ 失效：对 create_time 做了函数处理
SELECT * FROM `order` WHERE YEAR(create_time) = 2024;

-- ✅ 改写：对值做处理，不动索引列
SELECT * FROM `order` WHERE create_time >= '2024-01-01' AND create_time < '2025-01-01';

-- ❌ 失效：隐式类型转换（phone 是 VARCHAR，传入 INT）
SELECT * FROM user WHERE phone = 13800000000;

-- ✅ 改写：类型匹配
SELECT * FROM user WHERE phone = '13800000000';
```

### 场景二：LIKE 以通配符开头

```sql
-- ❌ 失效：前缀通配符，无法用 B+ 树定位起始
SELECT * FROM user WHERE name LIKE '%张';

-- ✅ 有效：后缀通配符，可以定位到 '张' 开头的范围
SELECT * FROM user WHERE name LIKE '张%';
```

> 💡 如果必须做前缀模糊查询，考虑引入 Elasticsearch 或全文索引。

### 场景三：OR 连接时有非索引列

```sql
-- 假设 name 有索引，email 没有
-- ❌ 失效：因为 email 无索引，OR 导致全表扫描
SELECT * FROM user WHERE name = '张三' OR email = 'a@b.com';

-- ✅ 改写：两列都加索引，或改写为 UNION
SELECT * FROM user WHERE name = '张三'
UNION
SELECT * FROM user WHERE email = 'a@b.com';
```

### 场景四：联合索引违反最左前缀

```sql
-- 索引: idx_name_age(name, age)
-- ❌ 失效：跳过了 name，直接查 age
SELECT * FROM user WHERE age = 25;
```

### 场景五：范围查询后的列失效

```sql
-- 索引: idx_age_status(age, status)
-- ❌ status 无法使用索引（age 做了范围查询后断开）
SELECT * FROM user WHERE age > 18 AND status = 1;

-- ✅ 改写：把等值列放前面，改索引为 idx_status_age(status, age)
SELECT * FROM user WHERE status = 1 AND age > 18;
```

### 场景六：负向查询（!=、NOT IN、NOT EXISTS）

```sql
-- ❌ 通常无法使用索引（范围太大，优化器认为全表更快）
SELECT * FROM user WHERE status != 0;
SELECT * FROM user WHERE id NOT IN (1, 2, 3);
```

### 失效场景速查表

| 失效原因 | 典型示例 | 解决思路 |
|---------|---------|---------|
| 索引列做函数 | `WHERE YEAR(dt) = 2024` | 改为范围比较 |
| 隐式类型转换 | VARCHAR 列 = 数字 | 保持类型一致 |
| LIKE 前缀通配符 | `LIKE '%xx'` | 改为后缀或用 ES |
| OR 有非索引列 | `col1=? OR 无索引col=?` | 补充索引或 UNION |
| 违反最左前缀 | 跳过联合索引首列 | 调整查询或索引顺序 |
| 范围查询后的列 | `age>18 AND status=1` | 等值列前置 |
| 负向查询 | `!=`、`NOT IN` | 改写为正向查询 |
| 字符集不一致 | JOIN 时两表字符集不同 | 统一字符集和排序规则 |

---

## 第四章：EXPLAIN 执行计划深度解读

### 4.1 基础用法

```sql
EXPLAIN SELECT * FROM user WHERE name = '张三';

-- MySQL 8.0 支持 EXPLAIN ANALYZE（显示实际执行时间）
EXPLAIN ANALYZE SELECT * FROM user WHERE name = '张三';
```

### 4.2 type 字段（最重要！代表访问类型）

type 从好到坏排序（性能依次降低）：

| type 值 | 含义 | 触发条件 |
|--------|------|---------|
| `system` | 表只有一行（系统表） | 特殊场景 |
| `const` | 通过主键或唯一索引，最多匹配一行 | `WHERE id = 1` |
| `eq_ref` | JOIN 时，被驱动表通过主键/唯一索引连接 | 一对一 JOIN |
| `ref` | 通过非唯一索引查找，可能多行 | `WHERE name = '张三'` |
| `range` | 索引范围扫描 | `WHERE age BETWEEN 18 AND 30` |
| `index` | 全索引扫描（比 ALL 好，但仍扫描整个索引树） | 覆盖索引但无 WHERE |
| `ALL` | 全表扫描（最差！） | 无索引或优化器放弃索引 |

> ⚠️ **生产 SQL 的 type 必须达到 `range` 及以上级别，出现 `ALL` 或 `index` 就需要立即优化。**

### 4.3 Extra 字段（非常重要）

| Extra 值 | 含义 | 好/坏 |
|---------|------|------|
| `Using index` | 覆盖索引，无需回表 | ✅ 好 |
| `Using where` | 从存储引擎取回数据后还需 Server 层过滤 | 中性 |
| `Using index condition` | 索引下推（ICP），部分过滤在引擎层完成 | ✅ 较好 |
| `Using filesort` | 需要额外排序，无法利用索引排序 | ⚠️ 需优化 |
| `Using temporary` | 使用了临时表（常见于 GROUP BY、DISTINCT） | ❌ 需优化 |
| `Using join buffer` | JOIN 时没用到索引，用了内存 buffer | ⚠️ 需优化 |

> ⚠️ **Extra 中出现 `Using filesort` 或 `Using temporary` 是高危信号，会严重影响并发性能，必须优化。**

### 4.4 其他关键字段

| 字段 | 说明 |
|------|------|
| `key` | 实际使用的索引名，为 NULL 说明没用到索引 |
| `possible_keys` | 候选索引列表 |
| `rows` | 优化器估计扫描行数，越小越好，超过 10 万要警惕 |
| `key_len` | 实际使用的索引长度，可推断联合索引命中了几列 |

### 4.5 索引下推（Index Condition Pushdown，ICP）

MySQL 5.6 引入，对联合索引查询有重要优化意义。

```sql
-- 索引: idx_name_age(name, age)
SELECT * FROM user WHERE name LIKE '张%' AND age = 25;
```

- **没有 ICP**：找到所有 `name LIKE '张%'` 的主键，全部回表，Server 层再过滤 `age = 25`
- **有 ICP**（Extra 显示 `Using index condition`）：遍历索引时同时检查 `age = 25`，不满足则跳过，大幅减少回表次数

---

## 第五章：慢 SQL 定位与排查工作流

### 5.1 慢查询日志配置

```sql
-- 查看当前慢查询配置
SHOW VARIABLES LIKE 'slow_query%';
SHOW VARIABLES LIKE 'long_query_time';

-- 动态开启（重启失效，生产临时使用）
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 1;                    -- 超过 1 秒记录
SET GLOBAL log_queries_not_using_indexes = ON;     -- 记录未用索引的 SQL
```

```ini
# 永久配置（写入 my.cnf）
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 1
log_queries_not_using_indexes = 1
```

### 5.2 mysqldumpslow 日志分析

```bash
# 按总执行时间排序，取前 10 条
mysqldumpslow -s t -t 10 /var/log/mysql/slow.log

# 按平均执行时间排序
mysqldumpslow -s at -t 10 /var/log/mysql/slow.log

# 按出现次数排序（找高频慢查询）
mysqldumpslow -s c -t 10 /var/log/mysql/slow.log
```

### 5.3 pt-query-digest（生产推荐）

```bash
# 安装
yum install percona-toolkit   # CentOS
apt install percona-toolkit   # Ubuntu

# 分析慢日志
pt-query-digest /var/log/mysql/slow.log

# 只看执行时间最长的 SQL
pt-query-digest --limit 10 /var/log/mysql/slow.log
```

输出报告包含：执行次数、总时间、平均时间、最大时间、锁等待时间、扫描行数等。

### 5.4 performance_schema 实时分析

```sql
-- 查看当前正在执行的 SQL 及耗时
SELECT * FROM performance_schema.events_statements_current
ORDER BY TIMER_WAIT DESC LIMIT 10;

-- 查看历史 SQL 执行统计（按总耗时排序）
SELECT DIGEST_TEXT,
       COUNT_STAR,
       AVG_TIMER_WAIT / 1000000000 AS avg_ms,
       SUM_TIMER_WAIT / 1000000000 AS total_ms
FROM performance_schema.events_statements_summary_by_digest
ORDER BY SUM_TIMER_WAIT DESC LIMIT 10;
```

### 5.5 SHOW PROCESSLIST：紧急情况定位

```sql
-- 查看当前所有连接和执行状态
SHOW FULL PROCESSLIST;

-- 找出执行时间超过 30 秒的查询
SELECT * FROM information_schema.PROCESSLIST
WHERE COMMAND != 'Sleep' AND TIME > 30
ORDER BY TIME DESC;
```

> 💡 线上出现数据库卡顿时，第一时间执行 `SHOW FULL PROCESSLIST`，查看是否有大量阻塞或 `Waiting for table metadata lock` 状态的连接。

### 5.6 慢 SQL 排查完整流程

1. 开启慢查询日志，收集慢 SQL（或从 APM/监控系统获取）
2. 用 `pt-query-digest` 聚合分析，按总耗时排序，找出「价值最高」的慢 SQL
3. 对目标 SQL 执行 `EXPLAIN`，重点看 `type`、`key`、`rows`、`Extra` 四列
4. 检查表结构（`SHOW CREATE TABLE`），确认是否有合适索引
5. 分析 WHERE、JOIN、ORDER BY 条件，验证索引是否按预期使用
6. 根据分析结果：补充索引 / 改写 SQL / 拆分复杂查询
7. 再次 `EXPLAIN` 验证优化效果，在测试环境压测对比
8. 上线后持续观察慢日志，确认问题解决

---

## 第六章：常见优化案例与 SQL 改写技巧

### 6.1 深分页问题（LIMIT offset 优化）

```sql
-- ❌ 慢：LIMIT 1000000, 10 会扫描并丢弃 100 万行数据
SELECT * FROM `order` ORDER BY id LIMIT 1000000, 10;

-- ✅ 方案一：子查询定位（覆盖索引找 id，再 JOIN）
SELECT o.* FROM `order` o
INNER JOIN (
  SELECT id FROM `order` ORDER BY id LIMIT 1000000, 10
) tmp ON o.id = tmp.id;

-- ✅ 方案二：游标翻页（记录上一页最后一个 id，性能最好）
SELECT * FROM `order` WHERE id > :last_max_id ORDER BY id LIMIT 10;
```

> 💡 游标翻页性能最好，适合「下一页」场景；子查询方式适合任意跳页场景。

### 6.2 ORDER BY 优化

```sql
-- 索引: idx_name_age(name, age)

-- ✅ 有效：排序列与索引一致，直接用索引排序，无 filesort
SELECT * FROM user WHERE name = '张三' ORDER BY age;

-- ❌ filesort：排序列不在索引中
SELECT * FROM user WHERE name = '张三' ORDER BY email;
```

ORDER BY 走索引的条件：排序列在索引中 + 排序方向与索引一致 + WHERE 等值条件列在排序列之前。

### 6.3 GROUP BY 优化

```sql
-- ❌ 触发 Using temporary + Using filesort
SELECT status, COUNT(*) FROM `order` GROUP BY status;

-- ✅ 给 status 加索引
CREATE INDEX idx_status ON `order`(status);

-- ✅ 如果不需要排序，加 ORDER BY NULL 避免 filesort
SELECT status, COUNT(*) FROM `order` GROUP BY status ORDER BY NULL;
```

### 6.4 JOIN 优化

| 优化点 | 说明 |
|-------|------|
| 小表驱动大表 | 把数据量小的表放在 JOIN 左侧（驱动表） |
| 被驱动表加索引 | JOIN 的关联列（ON 条件）必须在被驱动表上有索引 |
| 统一字符集 | 两表关联列字符集不同会导致隐式转换，索引失效 |
| 减少 JOIN 表数量 | 超过 3 张表的 JOIN 考虑拆分或在应用层组装 |
| 避免 SELECT * | 只查需要的列，减少数据传输量 |

```sql
-- ❌ 字符集不一致导致索引失效
-- a.user_id 是 utf8mb4，b.id 是 utf8
SELECT * FROM a JOIN b ON a.user_id = b.id;

-- ✅ 统一字符集
ALTER TABLE b CONVERT TO CHARACTER SET utf8mb4;
```

### 6.5 COUNT 优化

```sql
-- COUNT(*) / COUNT(1) / COUNT(id) 性能基本一致
-- COUNT(*) 包含 NULL，COUNT(列名) 不包含 NULL

-- ❌ 慢：InnoDB 的 COUNT(*) 每次都需要扫描
SELECT COUNT(*) FROM `order`;

-- ✅ 方案一：用选择性高的二级索引列（索引树更小）
SELECT COUNT(status) FROM `order`;  -- status 有索引

-- ✅ 方案二：用统计表缓存 COUNT 值（高并发场景）
-- ✅ 方案三：SHOW TABLE STATUS 的 Rows 字段（近似值）
```

### 6.6 IN / EXISTS 选择

```sql
-- 内表（子查询）数据量小，用 IN
SELECT * FROM `order` WHERE user_id IN (SELECT id FROM vip_user);

-- 外表数据量小，用 EXISTS
SELECT * FROM user u
WHERE EXISTS (SELECT 1 FROM `order` o WHERE o.user_id = u.id);
```

> 💡 MySQL 8.0 优化器已对两者做了趋同优化，现代版本差异不大，优先考虑可读性。

---

## 第七章：其他 MySQL 性能优化手段

### 7.1 数据类型选择

| 原则 | 说明 |
|------|------|
| 能用数字就不用字符串 | 状态字段用 `TINYINT` 不用 `VARCHAR` |
| 字段长度精准 | 按实际最大长度设置，不要无脑 `VARCHAR(255)` |
| 时间字段用 `DATETIME/TIMESTAMP` | 不要用字符串存时间 |
| 避免过多 NULL | 用默认值替代，`status` 默认 0 而非 NULL |
| 大字段单独存 | `TEXT/BLOB` 抽取到单独的详情表 |

### 7.2 Buffer Pool 调优

```sql
-- 查看 Buffer Pool 命中率（应该 > 99%）
SHOW STATUS LIKE 'Innodb_buffer_pool_read%';
-- 命中率 = 1 - Innodb_buffer_pool_reads / Innodb_buffer_pool_read_requests

-- 查看当前 Buffer Pool 大小
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';
```

> 💡 建议将 Buffer Pool 设置为服务器内存的 **60%~75%**。命中率低于 99% 说明内存不足。

### 7.3 定期维护

```sql
-- 更新表统计信息（优化器依赖统计信息做执行计划）
ANALYZE TABLE user;

-- 重建表（回收碎片空间）
OPTIMIZE TABLE user;

-- 查看表碎片率
SELECT TABLE_NAME, DATA_FREE / 1024 / 1024 AS data_free_mb
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'your_db' AND DATA_FREE > 0
ORDER BY data_free_mb DESC;
```

---

## 第八章：生产环境排查 SOP

### 8.1 线上数据库卡顿紧急处理

1. `SHOW FULL PROCESSLIST` → 找出 Time 最大、State 异常的线程
2. 确认是否有锁等待（`Waiting for table metadata lock`）
3. 查看活跃事务和锁等待详情
4. 必要时 `KILL <thread_id>` 终止阻塞线程（需 DBA 权限，谨慎操作）
5. 查看监控判断瓶颈类型（CPU / IO / 网络）
6. 查慢日志或 performance_schema，定位具体 SQL

```sql
-- 查看锁等待详情
SELECT
  r.trx_id               AS waiting_trx_id,
  r.trx_mysql_thread_id  AS waiting_thread,
  r.trx_query            AS waiting_query,
  b.trx_id               AS blocking_trx_id,
  b.trx_mysql_thread_id  AS blocking_thread
FROM information_schema.INNODB_LOCK_WAITS w
INNER JOIN information_schema.INNODB_TRX b ON b.trx_id = w.blocking_trx_id
INNER JOIN information_schema.INNODB_TRX r ON r.trx_id = w.requesting_trx_id;
```

### 8.2 索引优化快速决策树

遇到慢 SQL 时，按以下顺序检查：

1. `EXPLAIN` 看 `type` 是否为 `ALL` → 补充合适索引
2. `EXPLAIN` 看 `key` 是否为 `NULL` → 排查索引失效原因（见第三章）
3. `EXPLAIN` 看 `rows` 是否过大（> 10 万）→ 优化索引选择性
4. `Extra` 看是否有 `Using filesort` → 优化 ORDER BY，让其走索引
5. `Extra` 看是否有 `Using temporary` → 优化 GROUP BY / 子查询
6. rows 合理但仍慢 → 检查回表（是否需要覆盖索引）
7. SQL 本身无问题 → 检查锁、Buffer Pool、硬件资源

### 8.3 工具命令速查

| 目的 | 命令 |
|------|------|
| 查看表所有索引 | `SHOW INDEX FROM table_name;` |
| 查看索引选择性 | `SELECT COUNT(DISTINCT col)/COUNT(*) FROM t;` |
| 查看表统计信息 | `SHOW TABLE STATUS LIKE 'table_name';` |
| 强制使用某索引 | `SELECT * FROM t FORCE INDEX(idx_name) WHERE ...;` |
| 禁止使用某索引 | `SELECT * FROM t IGNORE INDEX(idx_name) WHERE ...;` |
| 查看优化器决策 | `SET optimizer_trace='enabled=on';` 然后执行 SQL，再查 `information_schema.OPTIMIZER_TRACE` |
| 查看表碎片 | `SELECT DATA_FREE FROM information_schema.TABLES WHERE TABLE_NAME='t';` |

### 8.4 推荐学习资源

**实践方向**
- 搭建本地 MySQL 8.0，对自己项目的核心表做 EXPLAIN 分析
- 用 sysbench 造百万级数据，亲自体验有无索引的性能差距
- 在测试库开启慢日志，跑一遍业务接口，收集并分析慢查询

**推荐书籍**
- 《MySQL 技术内幕：InnoDB 存储引擎》— 深入原理，必读
- 《高性能 MySQL》（第 4 版）— 实战优化经典

**在线工具**
- Percona Toolkit（`pt-query-digest`）— 生产慢日志分析利器
- MySQL Workbench 的 Visual Explain — 图形化执行计划

---

> 掌握索引原理 → 会看执行计划 → 能排查慢 SQL → 知其所以然
