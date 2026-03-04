# MySQL 锁机制专项学习手册

> 适合 Java 工程师 · 从原理到实战 · 含死锁排查 SOP · 高并发场景避坑指南

---

## 目录

1. [锁的整体分类体系](#第一章锁的整体分类体系)
2. [全局锁与表锁](#第二章全局锁与表锁)
3. [InnoDB 行锁详解](#第三章innodb-行锁详解)
4. [间隙锁与临键锁（幻读防护）](#第四章间隙锁与临键锁幻读防护)
5. [事务隔离级别与锁的关系](#第五章事务隔离级别与锁的关系)
6. [死锁：原理、排查与解决](#第六章死锁原理排查与解决)
7. [锁的监控与诊断](#第七章锁的监控与诊断)
8. [高并发实战场景与锁设计](#第八章高并发实战场景与锁设计)
9. [锁优化核心原则](#第九章锁优化核心原则)

---

## 第一章：锁的整体分类体系

### 1.1 为什么要有锁？

> 💡 **Java 类比**：MySQL 的锁 ≈ Java 并发包里的 `synchronized` / `ReentrantLock`。多个事务并发操作同一行数据时，没有锁就会出现脏读、丢失更新等并发问题。

锁的本质是**用串行化保护临界资源**，代价是并发度降低。MySQL 的锁设计目标是：**在保证数据正确性的前提下，让锁的粒度尽可能小、持有时间尽可能短**。

### 1.2 MySQL 锁的三层分类

```
MySQL 锁
├── 按粒度分
│   ├── 全局锁（Global Lock）        —— 锁整个数据库实例
│   ├── 表级锁（Table Lock）         —— 锁一张表
│   │   ├── 表锁（LOCK TABLES）
│   │   ├── 元数据锁（MDL Lock）
│   │   └── 意向锁（IS / IX）
│   └── 行级锁（Row Lock）           —— 锁一行或一个范围（InnoDB 独有）
│       ├── 记录锁（Record Lock）
│       ├── 间隙锁（Gap Lock）
│       └── 临键锁（Next-Key Lock）
│
├── 按模式分
│   ├── 共享锁 S（Shared Lock）      —— 读锁，多事务可同时持有
│   └── 排他锁 X（Exclusive Lock）   —— 写锁，独占，与任何锁互斥
│
└── 按加锁方式分
    ├── 自动加锁                      —— DML/SELECT FOR UPDATE 等自动触发
    └── 手动加锁                      —— LOCK TABLES、SELECT ... FOR SHARE
```

### 1.3 锁兼容矩阵

| | S（共享锁） | X（排他锁） |
|---|---|---|
| **S（共享锁）** | 兼容 ✅ | 冲突 ❌ |
| **X（排他锁）** | 冲突 ❌ | 冲突 ❌ |

> 读读不互斥，读写/写写互斥。这是所有并发控制的基础。

---

## 第二章：全局锁与表锁

### 2.1 全局锁

全局锁让整个数据库实例处于只读状态，所有的增删改、DDL 全部阻塞。

```sql
-- 加全局读锁（典型场景：全库逻辑备份）
FLUSH TABLES WITH READ LOCK;   -- 简称 FTWRL

-- 此时所有写操作被阻塞
UPDATE user SET status = 0;    -- 阻塞，等待全局锁释放

-- 释放全局锁
UNLOCK TABLES;
```

**为什么备份要加全局锁？**

不加锁备份时，如果备份到一半发生写入，备份文件中的表 A 是新数据、表 B 是旧数据，导致数据不一致。全局锁保证备份期间整个库的数据是同一个时间点的快照。

**更优方案：`--single-transaction`（不加锁备份 InnoDB）**

```bash
# mysqldump 加 --single-transaction 参数
# 利用 InnoDB 的 MVCC 快照，备份期间不阻塞写入
mysqldump --single-transaction -u root -p mydb > mydb.sql

# ⚠️ 仅对 InnoDB 有效，MyISAM 仍需全局锁
```

> 💡 **原理**：`--single-transaction` 在备份开始时开启一个事务，InnoDB 通过 MVCC 读取事务开始时的快照数据，整个备份期间的写入对备份不可见，无需加锁。

### 2.2 表级锁：LOCK TABLES

```sql
-- 手动加表级读锁（当前会话只读，其他会话可读不可写）
LOCK TABLES sys_user READ;

-- 手动加表级写锁（当前会话可读写，其他会话不可读不可写）
LOCK TABLES sys_user WRITE;

-- 释放所有表锁
UNLOCK TABLES;
```

**MyISAM 自动加表锁：**

```sql
-- MyISAM SELECT → 自动加表级读锁
SELECT * FROM myisam_table WHERE id = 1;

-- MyISAM UPDATE/INSERT/DELETE → 自动加表级写锁
UPDATE myisam_table SET name = 'x' WHERE id = 1;
```

> ⚠️ MyISAM 只有表锁，没有行锁，并发写性能极差。**InnoDB 是 MySQL 默认引擎，支持行锁，生产环境几乎不用 MyISAM。**

### 2.3 元数据锁（MDL Lock）

元数据锁（Metadata Lock）由 **MySQL Server 层自动管理**，开发者无需手动加锁，但必须了解，因为它是线上锁表问题的高频来源。

| 操作 | 自动加 MDL 类型 |
|------|----------------|
| SELECT / DML（增删改查） | MDL 读锁（共享） |
| ALTER TABLE / CREATE INDEX | MDL 写锁（独占） |

**经典线上故障场景：ALTER TABLE 被 MDL 读锁阻塞**

```
时间线：
T1: 事务 A 开始（SELECT * FROM user）           → 持有 MDL 读锁
T2: 事务 B 开始（SELECT * FROM user）           → 持有 MDL 读锁
T3: DBA 执行 ALTER TABLE user ADD COLUMN ...   → 申请 MDL 写锁，被 T1/T2 阻塞
T4: 事务 C 来了（SELECT * FROM user）           → 申请 MDL 读锁，被 T3 排在队列后阻塞
T5: 此后所有对 user 表的操作全部阻塞 → 表"假死"
```

```sql
-- 排查 MDL 锁阻塞（MySQL 5.7+）
SELECT * FROM performance_schema.metadata_locks
WHERE OBJECT_NAME = 'user'\G

-- 查看阻塞链（谁阻塞了谁）
SELECT
    r.trx_id             AS waiting_trx_id,
    r.trx_mysql_thread_id AS waiting_thread,
    r.trx_query          AS waiting_query,
    b.trx_id             AS blocking_trx_id,
    b.trx_mysql_thread_id AS blocking_thread,
    b.trx_query          AS blocking_query
FROM information_schema.innodb_lock_waits w
INNER JOIN information_schema.innodb_trx b ON b.trx_id = w.blocking_trx_id
INNER JOIN information_schema.innodb_trx r ON r.trx_id = w.requesting_trx_id;
```

**解决思路：**

1. 用 `SHOW PROCESSLIST` 找到持有 MDL 锁的长事务
2. `KILL <thread_id>` 干掉长事务
3. DDL 操作加超时限制，避免无限等待：

```sql
-- 设置 ALTER TABLE 等待 MDL 锁的超时时间（秒），超时后放弃而非一直等
ALTER TABLE sys_user
    WAIT 5                        -- 最多等 5 秒，超时返回错误
    ADD COLUMN extra VARCHAR(64);

-- 或者用 pt-online-schema-change / gh-ost 工具做无锁 DDL
```

### 2.4 意向锁（IS / IX）

意向锁是 InnoDB **表级别的锁**，由 InnoDB 自动维护，开发者无需关心，但理解它有助于理解行锁与表锁的配合。

| 行锁类型 | 自动加的意向锁 |
|---------|-------------|
| 行级共享锁 S | 表级意向共享锁 IS |
| 行级排他锁 X | 表级意向排他锁 IX |

**作用：** 让表锁和行锁的申请高效判断是否冲突，而不用遍历所有行。

```
没有意向锁：LOCK TABLES user WRITE 时，必须扫描每一行，看有没有行锁在占用。
有了意向锁：只需检查表上有没有 IS/IX 意向锁，O(1) 判断。
```

---

## 第三章：InnoDB 行锁详解

### 3.1 行锁的核心前提：基于索引

> ⚠️ **InnoDB 的行锁是加在索引上的，不是加在数据行上的。**
> 如果 WHERE 条件没走索引，InnoDB 会退化为锁全表！

```sql
-- 表结构
CREATE TABLE sys_user (
    id       BIGINT PRIMARY KEY,
    username VARCHAR(50),
    dept_id  INT,
    status   TINYINT,
    INDEX idx_dept_id(dept_id)
);

-- ✅ WHERE 走主键索引 → 精准行锁（只锁 id=100 这一行）
BEGIN;
SELECT * FROM sys_user WHERE id = 100 FOR UPDATE;
-- 只有 id=100 的行被锁，其他行不受影响

-- ✅ WHERE 走二级索引 → 锁二级索引 + 回表锁主键索引
BEGIN;
SELECT * FROM sys_user WHERE dept_id = 10 FOR UPDATE;
-- dept_id=10 的二级索引条目 + 对应的主键行都被锁

-- ❌ WHERE 没走索引 → 锁升级为全表锁！
BEGIN;
SELECT * FROM sys_user WHERE username = 'zhangsan' FOR UPDATE;
-- username 没有索引 → 全表扫描 → 给每一行都加 X 锁 → 等价于表锁
```

### 3.2 记录锁（Record Lock）

记录锁是最基础的行锁，锁住索引树上的**一条具体记录**。

```sql
-- 主键等值查询，命中记录 → 记录锁
BEGIN;
UPDATE sys_user SET status = 0 WHERE id = 100;
-- 仅锁住 id=100 这一条记录的主键索引节点

-- EXPLAIN 确认走主键索引
EXPLAIN UPDATE sys_user SET status = 0 WHERE id = 100;
-- type: const ✅
```

```sql
-- 验证记录锁：会话 A 锁住 id=100
-- 会话 A:
BEGIN;
SELECT * FROM sys_user WHERE id = 100 FOR UPDATE;

-- 会话 B：同一行，阻塞
SELECT * FROM sys_user WHERE id = 100 FOR UPDATE;  -- 阻塞等待

-- 会话 B：不同行，不阻塞
SELECT * FROM sys_user WHERE id = 101 FOR UPDATE;  -- 正常返回 ✅
```

### 3.3 共享锁 S 与排他锁 X 的加锁方式

```sql
-- 加共享锁（S Lock）：允许其他事务继续加 S 锁，但阻塞 X 锁
-- 语法：FOR SHARE（MySQL 8.0）或 LOCK IN SHARE MODE（兼容写法）
SELECT * FROM sys_user WHERE id = 100 FOR SHARE;
SELECT * FROM sys_user WHERE id = 100 LOCK IN SHARE MODE;

-- 加排他锁（X Lock）：阻塞其他事务的 S 锁和 X 锁
SELECT * FROM sys_user WHERE id = 100 FOR UPDATE;

-- DML 自动加 X 锁（无需手写 FOR UPDATE）
UPDATE sys_user SET status = 0 WHERE id = 100;  -- 自动加 X 锁
DELETE FROM sys_user WHERE id = 100;            -- 自动加 X 锁
INSERT INTO sys_user VALUES (...);              -- 自动加 X 锁（插入意向锁）
```

**锁的持有时间：**

```sql
-- InnoDB 行锁随事务生命周期
BEGIN;
UPDATE sys_user SET status = 0 WHERE id = 100;  -- 加 X 锁
-- ... 其他操作 ...
COMMIT;   -- 事务提交，X 锁释放
-- 或
ROLLBACK; -- 事务回滚，X 锁释放

-- ⚠️ 锁在事务提交/回滚前一直持有，事务越长，锁持有时间越长，并发越差
```

---

## 第四章：间隙锁与临键锁（幻读防护）

### 4.1 为什么需要间隙锁？

**幻读**：同一事务内，两次相同的范围查询返回的行数不同（中间有其他事务插入了新行）。

```sql
-- 隔离级别：REPEATABLE READ（MySQL 默认）
-- 会话 A：
BEGIN;
SELECT * FROM sys_user WHERE dept_id = 10;  -- 返回 5 行

-- 会话 B：
INSERT INTO sys_user(id, dept_id) VALUES(999, 10);  -- 插入新行
COMMIT;

-- 会话 A 再次查询：
SELECT * FROM sys_user WHERE dept_id = 10;  -- 如果出现 6 行 → 幻读
```

InnoDB 在 **REPEATABLE READ** 下通过**间隙锁（Gap Lock）**防止幻读。

### 4.2 间隙锁（Gap Lock）

间隙锁锁住的是**索引记录之间的间隙**，阻止其他事务在该间隙内插入数据。

```sql
-- 假设 dept_id 索引上有以下值：5, 10, 15, 20
-- 间隙定义：(-∞,5)  (5,10)  (10,15)  (15,20)  (20,+∞)

-- 会话 A：范围查询加锁
BEGIN;
SELECT * FROM sys_user WHERE dept_id BETWEEN 10 AND 15 FOR UPDATE;
-- 锁住：
--   记录锁：dept_id=10、dept_id=15 的行
--   间隙锁：(5,10) 和 (15,20) 之间的间隙（防止插入 dept_id=11/12/14 等）

-- 会话 B：尝试在间隙内插入，阻塞
INSERT INTO sys_user(id, dept_id) VALUES(888, 12);  -- 阻塞！dept_id=12 在间隙内

-- 会话 B：在间隙外插入，不阻塞
INSERT INTO sys_user(id, dept_id) VALUES(889, 3);   -- 正常 ✅ dept_id=3 不在锁定范围
```

> ⚠️ **间隙锁只在 REPEATABLE READ 隔离级别下生效**。READ COMMITTED 下没有间隙锁，幻读防护依赖 MVCC 快照读。

**间隙锁的特殊性：**

```sql
-- 间隙锁之间不互斥！两个事务可以同时持有同一间隙的间隙锁
-- 会话 A：
BEGIN;
SELECT * FROM sys_user WHERE dept_id = 12 FOR UPDATE;
-- dept_id=12 不存在，加间隙锁 (10, 15)

-- 会话 B：
BEGIN;
SELECT * FROM sys_user WHERE dept_id = 13 FOR UPDATE;
-- dept_id=13 不存在，加间隙锁 (10, 15)
-- 两个事务都成功，间隙锁不互斥

-- 但此时 A 和 B 如果都尝试插入 dept_id=11 → 死锁！（见第六章）
```

### 4.3 临键锁（Next-Key Lock）

临键锁 = 记录锁 + 左侧间隙锁，是 InnoDB 在 REPEATABLE READ 下**默认的行锁形式**。

```
临键锁的范围是左开右闭区间：(前一个值, 当前值]

dept_id 索引值：5, 10, 15, 20
临键锁范围：
  (-∞, 5]
  (5, 10]
  (10, 15]
  (15, 20]
  (20, +∞)  ← 最右侧是纯间隙锁（supremum 伪记录）
```

```sql
-- 等值查询命中记录 → 临键锁退化为记录锁
SELECT * FROM sys_user WHERE dept_id = 10 FOR UPDATE;
-- 退化：只加记录锁，不加间隙锁（因为是等值且命中）

-- 等值查询未命中（查询值不存在）→ 退化为间隙锁
SELECT * FROM sys_user WHERE dept_id = 12 FOR UPDATE;
-- dept_id=12 不存在，退化为间隙锁 (10, 15)

-- 范围查询 → 标准临键锁
SELECT * FROM sys_user WHERE dept_id > 10 FOR UPDATE;
-- 锁住：(10, 15]、(15, 20]、(20, +∞)
```

### 4.4 锁退化规则总结

| 查询场景 | 索引类型 | 锁类型 |
|---------|---------|--------|
| 等值查询，命中唯一索引 | 主键 / UNIQUE | 记录锁（精准锁单行） |
| 等值查询，命中普通索引 | 普通索引 | 临键锁（含左侧间隙） |
| 等值查询，未命中任何记录 | 任意 | 间隙锁 |
| 范围查询 | 任意 | 临键锁（锁范围内所有区间） |
| 全表扫描（无索引） | 无 | 全表 X 锁（等价表锁） |

---

## 第五章：事务隔离级别与锁的关系

### 5.1 四种隔离级别回顾

| 隔离级别 | 脏读 | 不可重复读 | 幻读 | 锁机制 |
|---------|------|-----------|------|--------|
| READ UNCOMMITTED | 可能 | 可能 | 可能 | 无锁读 |
| READ COMMITTED | 不可能 | 可能 | 可能 | 快照读（语句级） |
| REPEATABLE READ（默认） | 不可能 | 不可能 | 基本不可能 | 快照读（事务级）+ 间隙锁 |
| SERIALIZABLE | 不可能 | 不可能 | 不可能 | 所有读加 S 锁，串行化 |

### 5.2 MVCC：快照读的底层原理

InnoDB 通过 **MVCC（Multi-Version Concurrency Control，多版本并发控制）** 让普通 SELECT 不加锁就能读到一致性快照。

#### 图1：每行数据的隐藏字段

```
你以为的一行数据：
┌─────┬──────────┬─────────┐
│ id  │ username │ dept_id │
├─────┼──────────┼─────────┤
│  1  │  张三    │   10    │
└─────┴──────────┴─────────┘

InnoDB 实际存的一行数据：
┌─────┬──────────┬─────────┬──────────┬──────────┐
│ id  │ username │ dept_id │  trx_id  │ roll_ptr │
├─────┼──────────┼─────────┼──────────┼──────────┤
│  1  │  张三    │   10    │   1000   │  ──────────→ undo log 历史版本
└─────┴──────────┴─────────┴──────────┴──────────┘
                                ↑              ↑
                          最后一次修改      指向上一个
                          这行的事务ID      历史版本
```

#### 图2：版本链（undo log）

每次修改不是覆盖旧数据，而是把旧版本存进 undo log，新旧版本用链表串起来：

```
现在的数据行（最新版本）
┌─────┬──────────┬─────────┬──────────┬──────────┐
│ id  │ username │ dept_id │  trx_id  │ roll_ptr │
│  1  │  张三    │   30    │   1003   │    ↓     │
└─────┴──────────┴─────────┴──────────┴──────────┘
                                            │
                                    ┌───────▼──────────────────────┐
                                    │  undo log 版本2              │
                                    │  dept_id=20  trx_id=1002  ↓ │
                                    └──────────────────────────────┘
                                                            │
                                                   ┌────────▼─────────────────────┐
                                                   │  undo log 版本1              │
                                                   │  dept_id=10  trx_id=1000  ↓ │
                                                   └──────────────────────────────┘
                                                                           │
                                                                         NULL（最老版本）

时间线：
  trx_id=1000  INSERT 张三，dept_id=10
  trx_id=1002  UPDATE dept_id=10 → 20
  trx_id=1003  UPDATE dept_id=20 → 30   ← 当前最新值
```

#### 图3：ReadView 决定你能看到哪个版本

事务开始时，MySQL 生成一张"快照名单"，记录当时哪些事务还没提交：

```
假设现在系统里有这些事务：

  事务100 ──已提交
  事务200 ──进行中──┐  快照名单（ReadView）
  事务300 ──进行中──┘  记录：[200, 300] 还未提交
                       min_trx_id=200，max_trx_id=401（下一个新事务起点）

你是 事务250，读取数据时拿版本链上每个 trx_id 来判断：

版本的 trx_id    判断规则                              能否看到？
────────────────────────────────────────────────────────────────
trx_id = 100    100 < min(200)，已提交                 ✅ 可以看
trx_id = 150    150 < min(200)，已提交                 ✅ 可以看
trx_id = 200    200 在名单[200,300]里，未提交           ❌ 看不到
trx_id = 300    300 在名单[200,300]里，未提交           ❌ 看不到
trx_id = 250    250 就是我自己改的                     ✅ 可以看
trx_id = 450    450 > max(401)，快照生成后才出现的事务  ❌ 看不到
```

#### 图4：完整快照读流程

```
事务A（trx_id=500）执行：SELECT dept_id FROM sys_user WHERE id = 1;

情况1：最新版本由已提交事务写入
  ┌─── 最新版本：dept_id=30，trx_id=1003 ───┐
  │    1003 < min(200)，已提交              │
  │    ✅ 直接返回 dept_id=30               │
  └─────────────────────────────────────────┘

情况2：最新版本由未提交事务写入
  ┌─── 最新版本：dept_id=30，trx_id=250 ────┐
  │    250 在名单[200,300]里，未提交 ❌      │
  │    沿 roll_ptr 找上一个版本             │
  └──────────────────┬──────────────────────┘
                     ↓
  ┌─── 上一个版本：dept_id=20，trx_id=100 ──┐
  │    100 < min(200)，已提交               │
  │    ✅ 返回 dept_id=20                   │
  └─────────────────────────────────────────┘
```

#### 图5：RC 与 RR 的本质区别

```
时间轴 ───────────────────────────────────────────────────────▶

事务B：  BEGIN ── UPDATE dept_id=99 ── COMMIT
                        ↑
事务A：  BEGIN ── SELECT ────────────────────── SELECT

┌──────────────────────────────────────────────────────────┐
│  READ COMMITTED（RC）                                    │
│  每次 SELECT 都重新生成 ReadView（语句级快照）            │
│  第一次 SELECT：B未提交，看到旧值 dept_id=10             │
│  第二次 SELECT：B已提交，看到新值 dept_id=99  ← 不可重复读│
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│  REPEATABLE READ（RR，MySQL 默认）                       │
│  BEGIN 时生成一次 ReadView，整个事务期间不再变化          │
│  第一次 SELECT：看到旧值 dept_id=10                      │
│  第二次 SELECT：还是旧值 dept_id=10  ✅ 可重复读         │
└──────────────────────────────────────────────────────────┘
```

```
每一行数据在 InnoDB 中隐藏了两个字段：
  trx_id：最近一次修改该行的事务 ID
  roll_ptr：指向 undo log 中该行的历史版本链

MVCC 读取逻辑：
  1. 事务开始时生成 ReadView（快照）
  2. 读取数据行时，检查 trx_id 是否在 ReadView 可见范围内
  3. 不可见 → 顺着 roll_ptr 找历史版本，直到找到可见版本

一句话总结：
  MVCC = 每行存版本链 + 事务开始时拍快照（ReadView）
         读数据时顺着版本链找"对我可见的最新版本"
         → 读不加锁，写不阻读，并发性能极大提升
```

```sql
-- 快照读（普通 SELECT，走 MVCC，不加锁）
SELECT * FROM sys_user WHERE id = 100;   -- 读历史快照，不阻塞

-- 当前读（SELECT FOR UPDATE / DML，读最新数据，加锁）
SELECT * FROM sys_user WHERE id = 100 FOR UPDATE;  -- 加 X 锁，读最新
UPDATE sys_user SET status = 0 WHERE id = 100;     -- 加 X 锁，读最新
```

### 5.3 READ COMMITTED vs REPEATABLE READ 锁行为差异

```sql
-- ====== READ COMMITTED ======
-- 特点：每次 SQL 语句开始时重新生成 ReadView（语句级快照）
-- 没有间隙锁，只有记录锁

-- 会话 A（RC 隔离级别）：
BEGIN;
SELECT * FROM sys_user WHERE dept_id = 10;  -- 快照：当前时刻

-- 会话 B 插入并提交：
INSERT INTO sys_user(id, dept_id) VALUES(999, 10); COMMIT;

-- 会话 A 再次查询：
SELECT * FROM sys_user WHERE dept_id = 10;  -- 能看到 B 插入的行（不可重复读）
```

```sql
-- ====== REPEATABLE READ（默认）======
-- 特点：事务开始时生成 ReadView，整个事务期间快照不变（事务级快照）
-- 有间隙锁，防止幻读

-- 会话 A（RR 隔离级别）：
BEGIN;
SELECT * FROM sys_user WHERE dept_id = 10;  -- 快照：事务开始时刻

-- 会话 B 插入并提交：
INSERT INTO sys_user(id, dept_id) VALUES(999, 10); COMMIT;

-- 会话 A 再次查询（快照读）：
SELECT * FROM sys_user WHERE dept_id = 10;  -- 看不到 B 插入的行 ✅（可重复读）

-- 但用当前读就能看到！
SELECT * FROM sys_user WHERE dept_id = 10 FOR UPDATE;  -- 能看到 B 插入的行（当前读）
```

> 💡 **RR 级别下的幻读防护：**
> - **快照读（普通 SELECT）**：MVCC 保证，看不到其他事务新插入的行。
> - **当前读（FOR UPDATE / DML）**：间隙锁保证，直接阻止其他事务插入到锁定范围。

---

## 第六章：死锁：原理、排查与解决

### 6.1 死锁的四个必要条件

```
1. 互斥：资源同时只能被一个事务持有（锁的本质）
2. 持有并等待：事务持有已有锁，同时等待其他事务释放锁
3. 不可抢占：已获得的锁不能被强制剥夺
4. 循环等待：事务 A 等 B，B 等 A，形成环
```

### 6.2 经典死锁场景

**场景一：两事务互相等待对方的行锁**

```sql
-- 会话 A：
BEGIN;
UPDATE sys_user SET status = 0 WHERE id = 100;  -- 锁住 id=100

-- 会话 B：
BEGIN;
UPDATE sys_user SET status = 0 WHERE id = 200;  -- 锁住 id=200

-- 会话 A：
UPDATE sys_user SET status = 0 WHERE id = 200;  -- 等待 B 释放 id=200 的锁

-- 会话 B：
UPDATE sys_user SET status = 0 WHERE id = 100;  -- 等待 A 释放 id=100 的锁
-- → 死锁！MySQL 自动选一个事务回滚（代价小的那个）
```

**场景二：间隙锁 + 插入死锁（高频！）**

```sql
-- dept_id 索引值：5, 10, 15

-- 会话 A：
BEGIN;
SELECT * FROM sys_user WHERE dept_id = 12 FOR UPDATE;
-- dept_id=12 不存在，加间隙锁 (10, 15)

-- 会话 B：
BEGIN;
SELECT * FROM sys_user WHERE dept_id = 13 FOR UPDATE;
-- dept_id=13 不存在，加间隙锁 (10, 15)
-- ✅ 间隙锁不互斥，A、B 都持有 (10,15) 的间隙锁

-- 会话 A 尝试插入：
INSERT INTO sys_user(id, dept_id) VALUES(101, 12);
-- 需要插入意向锁，被 B 的间隙锁阻塞，等待 B 释放

-- 会话 B 也尝试插入：
INSERT INTO sys_user(id, dept_id) VALUES(102, 13);
-- 需要插入意向锁，被 A 的间隙锁阻塞，等待 A 释放
-- → 死锁！
```

**场景三：不同表加锁顺序不一致**

```sql
-- 会话 A（先锁 user 再锁 order）：
BEGIN;
SELECT * FROM sys_user WHERE id = 1 FOR UPDATE;
SELECT * FROM `order` WHERE user_id = 1 FOR UPDATE;

-- 会话 B（先锁 order 再锁 user，顺序相反）：
BEGIN;
SELECT * FROM `order` WHERE user_id = 1 FOR UPDATE;
SELECT * FROM sys_user WHERE id = 1 FOR UPDATE;
-- → 死锁！
```

### 6.3 死锁排查 SOP

**Step 1：查看最近一次死锁信息**

```sql
-- 显示最近一次死锁详情（InnoDB 状态）
SHOW ENGINE INNODB STATUS\G
-- 找 LATEST DETECTED DEADLOCK 段
-- 内容包括：
--   死锁的两个事务各自持有的锁
--   各自在等待什么锁
--   哪个事务被选为 victim（回滚方）
```

**Step 2：解读 INNODB STATUS 死锁信息**

```
------------------------
LATEST DETECTED DEADLOCK
------------------------
*** (1) TRANSACTION:
TRANSACTION 421, ACTIVE 5 sec starting index read
MySQL thread id 10, OS thread handle 12345, query id 100
UPDATE sys_user SET status=0 WHERE id=200        ← 事务1正在执行的 SQL
*** (1) HOLDS THE LOCK(S):
RECORD LOCKS ... index PRIMARY of table `sys_user` trx id 421
lock_mode X locks rec but not gap              ← 持有 id=100 的记录锁
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS ... index PRIMARY of table `sys_user` trx id 421
lock_mode X locks rec but not gap              ← 在等 id=200 的记录锁

*** (2) TRANSACTION:
TRANSACTION 422, ACTIVE 3 sec starting index read
*** (2) HOLDS THE LOCK(S):          ← 持有 id=200 的记录锁
*** (2) WAITING FOR THIS LOCK TO BE GRANTED:  ← 在等 id=100 的记录锁

*** WE ROLL BACK TRANSACTION (2)    ← MySQL 选择回滚事务2
```

**Step 3：开启死锁日志持久化**

```sql
-- 将死锁信息写入错误日志（生产环境建议开启）
SET GLOBAL innodb_print_all_deadlocks = ON;

-- 查看当前锁等待情况
SELECT * FROM performance_schema.data_lock_waits\G

-- 查看当前所有锁
SELECT * FROM performance_schema.data_locks
WHERE LOCK_STATUS = 'WAITING'\G
```

### 6.4 死锁解决方案

| 策略 | 说明 | 适用场景 |
|------|------|---------|
| 固定加锁顺序 | 所有业务代码按相同顺序访问资源 | 多表操作、批量更新 |
| 缩短事务 | 减少事务持有锁的时间 | 通用 |
| 减小锁粒度 | 精确 WHERE 条件，避免范围锁 | 范围查询场景 |
| 降低隔离级别 | RC 无间隙锁，消除间隙锁死锁 | 允许幻读的场景 |
| 一次性锁 | `SELECT ... FOR UPDATE` 提前锁住所有资源 | 转账等关键场景 |
| 应用层重试 | 捕获死锁异常（错误码 1213）自动重试 | 所有涉及并发写的接口 |

```java
// Java 应用层死锁重试（Spring + MyBatis 场景）
@Transactional
public void updateWithRetry(Long id) {
    int maxRetry = 3;
    for (int i = 0; i < maxRetry; i++) {
        try {
            userMapper.updateStatus(id);
            return;
        } catch (DataAccessException e) {
            // MySQL 死锁错误码 1213
            if (e.getMessage().contains("Deadlock") && i < maxRetry - 1) {
                log.warn("死锁重试，第 {} 次", i + 1);
                continue;
            }
            throw e;
        }
    }
}
```

```sql
-- 设置锁等待超时（避免长时间等待，超时抛异常让应用重试）
SET innodb_lock_wait_timeout = 5;  -- 等待锁超过 5 秒抛异常，默认 50 秒

-- 全局设置
SET GLOBAL innodb_lock_wait_timeout = 5;
```

---

## 第七章：锁的监控与诊断

### 7.1 实时查看锁等待

```sql
-- 查看当前所有锁等待关系（MySQL 8.0+，performance_schema 需开启）
SELECT
    r.THREAD_ID           AS waiting_thread,
    r.OBJECT_NAME         AS table_name,
    r.LOCK_TYPE           AS lock_type,
    r.LOCK_MODE           AS waiting_lock_mode,
    b.THREAD_ID           AS blocking_thread,
    b.LOCK_MODE           AS blocking_lock_mode
FROM performance_schema.data_lock_waits w
JOIN performance_schema.data_locks r ON r.ENGINE_LOCK_ID = w.REQUESTING_ENGINE_LOCK_ID
JOIN performance_schema.data_locks b ON b.ENGINE_LOCK_ID = w.BLOCKING_ENGINE_LOCK_ID;
```

### 7.2 查看当前活跃事务

```sql
-- 查看所有活跃事务（包括持有锁的事务）
SELECT
    trx_id,
    trx_state,                          -- RUNNING / LOCK WAIT / ROLLING BACK
    trx_started,                        -- 事务开始时间
    trx_mysql_thread_id,               -- 线程 ID（用于 KILL）
    trx_query,                          -- 当前正在执行的 SQL
    trx_rows_locked,                    -- 锁住的行数
    trx_lock_memory_bytes               -- 锁占用内存
FROM information_schema.innodb_trx
ORDER BY trx_started ASC;              -- 按开始时间排，老事务在前

-- 找出长时间未提交的事务（超过 60 秒）
SELECT * FROM information_schema.innodb_trx
WHERE TIMESTAMPDIFF(SECOND, trx_started, NOW()) > 60;
```

### 7.3 查看锁等待并找到阻塞源头

```sql
-- 完整锁等待链路：谁在等谁（MySQL 8.0）
SELECT
    waiting.trx_id           AS waiting_trx_id,
    waiting.trx_mysql_thread_id AS waiting_pid,
    waiting.trx_query         AS waiting_sql,
    blocking.trx_id          AS blocking_trx_id,
    blocking.trx_mysql_thread_id AS blocking_pid,
    blocking.trx_query        AS blocking_sql,
    blocking.trx_started      AS blocking_start_time,
    TIMESTAMPDIFF(SECOND, blocking.trx_started, NOW()) AS blocking_seconds
FROM information_schema.innodb_lock_waits  w
JOIN information_schema.innodb_trx waiting  ON waiting.trx_id  = w.requesting_trx_id
JOIN information_schema.innodb_trx blocking ON blocking.trx_id = w.blocking_trx_id
ORDER BY blocking_seconds DESC;
```

### 7.4 SHOW PROCESSLIST 快速诊断

```sql
-- 查看所有连接（包括状态）
SHOW FULL PROCESSLIST;

-- 重点关注 State 列：
--   "Waiting for table metadata lock"  → MDL 锁等待
--   "Waiting for this lock to be granted" → 行锁等待
--   "Sleep" 且 Time 很大 → 长事务未提交，可能持有锁

-- 干掉阻塞的长事务
KILL <thread_id>;
```

### 7.5 锁监控关键指标

```sql
-- 查看锁相关状态变量
SHOW STATUS LIKE 'innodb_row_lock%';
-- Innodb_row_lock_current_waits  当前正在等待的锁数量（>0 说明有锁竞争）
-- Innodb_row_lock_time_avg       平均等锁时间（毫秒），高了说明锁竞争激烈
-- Innodb_row_lock_waits          历史总等锁次数
-- Innodb_row_lock_time           历史总等锁时间

-- 查看死锁计数
SHOW STATUS LIKE 'innodb_deadlocks';  -- MySQL 8.0.34+
```

---

## 第八章：高并发实战场景与锁设计

### 8.1 场景一：库存扣减（超卖问题）

**问题复现：**

```sql
-- ❌ 错误写法：先查后改，高并发下会超卖
-- 伪代码：
-- 1. SELECT stock FROM product WHERE id = 1;  → stock = 1
-- 2. if stock > 0: UPDATE product SET stock = stock - 1 WHERE id = 1;
-- 并发时：两个请求都读到 stock=1，都通过检查，stock 减两次变成 -1
```

**方案一：数据库层防超卖（WHERE 加库存判断）**

```sql
-- ✅ UPDATE 的 WHERE 条件中判断库存充足
-- 利用 UPDATE 的原子性：UPDATE 本身会加 X 锁，并发安全
UPDATE product
SET stock = stock - 1
WHERE id = 1
  AND stock > 0;     -- 库存不足时 affected rows = 0，应用层判断并返回"库存不足"

-- Java 层：
int affected = productMapper.decreaseStock(productId);
if (affected == 0) {
    throw new BusinessException("库存不足");
}
```

**方案二：SELECT FOR UPDATE 显式锁（需要读出库存做业务逻辑判断时）**

```sql
BEGIN;
-- 先加 X 锁读出库存
SELECT stock FROM product WHERE id = 1 FOR UPDATE;
-- 业务逻辑判断（Java 层）
-- 扣减
UPDATE product SET stock = stock - 1 WHERE id = 1;
COMMIT;

-- ⚠️ 注意：FOR UPDATE 是串行化的，高并发下吞吐会降低
-- 适合秒杀等"宁可排队也不超卖"的场景
```

**方案三：乐观锁（version 版本号，低竞争场景）**

```sql
-- 表增加 version 字段
ALTER TABLE product ADD COLUMN version INT DEFAULT 0;

-- 更新时带版本号 CAS
UPDATE product
SET stock = stock - 1, version = version + 1
WHERE id = 1
  AND stock > 0
  AND version = #{version};   -- 版本号不匹配（被别人改过）→ affected=0 → 重试

-- 适合：竞争不激烈、允许少量重试的场景
-- 不适合：秒杀（大量重试会导致 CPU 飙高）
```

### 8.2 场景二：转账（多行更新的死锁防护）

```sql
-- ❌ 危险写法：加锁顺序不固定，两个线程反向转账时死锁
-- 线程1（A→B）：先锁 A，再锁 B
-- 线程2（B→A）：先锁 B，再锁 A

-- ✅ 安全写法：固定加锁顺序（总是按 id 从小到大加锁）
BEGIN;

-- 先锁 id 较小的账户（固定顺序，避免死锁）
SELECT balance FROM account WHERE id = LEAST(from_id, to_id)   FOR UPDATE;
SELECT balance FROM account WHERE id = GREATEST(from_id, to_id) FOR UPDATE;

-- 扣款和入账
UPDATE account SET balance = balance - 100 WHERE id = from_id;
UPDATE account SET balance = balance + 100 WHERE id = to_id;

COMMIT;
```

```sql
-- 更简洁：一次性用 IN 锁住两行（InnoDB 会按主键顺序加锁，天然有序）
BEGIN;
SELECT * FROM account WHERE id IN (from_id, to_id) FOR UPDATE;
-- InnoDB 按主键顺序加锁，不会死锁

UPDATE account SET balance = balance - 100 WHERE id = from_id;
UPDATE account SET balance = balance + 100 WHERE id = to_id;
COMMIT;
```

### 8.3 场景三：防重复提交（唯一索引 + 锁）

```sql
-- 利用唯一索引做幂等控制（不需要手动加锁）
CREATE TABLE order_dedup (
    request_id VARCHAR(64) PRIMARY KEY,   -- 请求唯一 ID
    order_id   BIGINT,
    create_time DATETIME
);

-- 业务代码：
-- 1. INSERT INTO order_dedup(request_id, order_id) VALUES(?, ?)
-- 2. 如果 INSERT 报 Duplicate key（错误码 1062）→ 幂等处理，返回已有订单
-- 3. 唯一索引保证并发安全，无需手动加锁
```

### 8.4 场景四：SELECT FOR UPDATE 导致的锁范围扩大问题

```sql
-- 场景：查不存在的记录加锁，不小心触发间隙锁，阻塞大量插入

-- 表 sys_user 中 dept_id 索引有值：10, 20, 30

-- ❌ 问题写法：查一个不存在的 dept_id
BEGIN;
SELECT * FROM sys_user WHERE dept_id = 15 FOR UPDATE;
-- dept_id=15 不存在，加间隙锁 (10, 20)
-- 此后所有往 dept_id ∈ (10,20) 的插入操作全部阻塞！

-- ✅ 改进：如果不需要防幻读，降到 READ COMMITTED
-- RC 下没有间隙锁，FOR UPDATE 只加记录锁
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
BEGIN;
SELECT * FROM sys_user WHERE dept_id = 15 FOR UPDATE;
-- 不存在记录 → 不加任何锁（RC 下没有间隙锁）
COMMIT;
```

---

## 第九章：锁优化核心原则

### 9.1 七条锁优化原则

| # | 原则 | 说明 |
|---|------|------|
| 1 | **缩短事务** | 事务内只做必须的操作，不做 RPC、不做复杂计算，减少锁持有时间 |
| 2 | **WHERE 必须走索引** | 无索引 → 全表 X 锁，比表锁更危险（还持有时间更长） |
| 3 | **固定加锁顺序** | 多表多行操作时，全局约定加锁顺序，消除死锁的循环等待 |
| 4 | **不在事务内做耗时操作** | 网络请求、文件 IO、大计算放在事务外，避免长事务持锁 |
| 5 | **用乐观锁替代悲观锁** | 低竞争场景用 version 号做 CAS，避免锁等待 |
| 6 | **降低隔离级别** | 业务允许时用 RC，消除间隙锁，提升并发 |
| 7 | **监控长事务** | 定期查 `innodb_trx`，及时 KILL 超时未提交的事务 |

### 9.2 锁粒度选择决策树

```
需要控制并发？
├── 只需要防止并发读写同一行
│     └── InnoDB 行锁（默认）→ UPDATE/DELETE 自动加，或 SELECT FOR UPDATE
│
├── 需要防止幻读（范围内插入新行）
│     ├── REPEATABLE READ + FOR UPDATE → 间隙锁自动生效
│     └── 应用层幂等（唯一索引）→ 更轻量
│
├── 需要备份一致性
│     ├── InnoDB → --single-transaction（MVCC 快照，不加锁）
│     └── MyISAM → FLUSH TABLES WITH READ LOCK（必须加锁）
│
└── 需要 DDL 不阻塞业务
      └── pt-online-schema-change / gh-ost 工具，或 ALGORITHM=INPLACE
```

### 9.3 常见锁问题速查表

| 现象 | 原因 | 排查命令 | 解决 |
|------|------|---------|------|
| 接口响应超时，DB 无明显慢 SQL | 锁等待 | `SHOW PROCESSLIST` | KILL 长事务 |
| `Using join buffer` 且行数极大 | 被驱动表无索引，退化 BNL | `EXPLAIN` 分析 | 给 ON 列加索引 |
| ALTER TABLE 卡住 | MDL 写锁等待 | `performance_schema.metadata_locks` | KILL 持有 MDL 读锁的长事务 |
| 死锁报错（1213） | 循环等待 | `SHOW ENGINE INNODB STATUS` | 固定加锁顺序 + 应用层重试 |
| 大量 `Waiting for this lock` | 热点行锁竞争 | `data_lock_waits` | 拆分热点行 / 队列削峰 |
| 插入被阻塞但查询看不到冲突行 | 间隙锁 | `data_locks` 看 LOCK_MODE=X,GAP | 降 RC 或优化查询条件 |

---

## 附录：锁相关 SQL 速查

```sql
-- 1. 查看当前等锁情况
SELECT * FROM performance_schema.data_lock_waits\G

-- 2. 查看所有活跃事务
SELECT trx_id, trx_state, trx_started, trx_mysql_thread_id, trx_query
FROM information_schema.innodb_trx ORDER BY trx_started;

-- 3. 查看最近死锁详情
SHOW ENGINE INNODB STATUS\G

-- 4. 查看所有锁（MySQL 8.0）
SELECT ENGINE_LOCK_ID, OBJECT_NAME, INDEX_NAME, LOCK_TYPE, LOCK_MODE, LOCK_STATUS
FROM performance_schema.data_locks;

-- 5. 查看 MDL 锁
SELECT * FROM performance_schema.metadata_locks WHERE OBJECT_NAME = 'your_table';

-- 6. 查看行锁统计
SHOW STATUS LIKE 'innodb_row_lock%';

-- 7. 设置锁等待超时
SET innodb_lock_wait_timeout = 5;

-- 8. 开启死锁日志
SET GLOBAL innodb_print_all_deadlocks = ON;

-- 9. 查看当前隔离级别
SELECT @@transaction_isolation;

-- 10. KILL 阻塞线程
KILL <thread_id>;  -- thread_id 从 SHOW PROCESSLIST 获取
```
