# MySQL 锁机制实战训练手册

> 用真实并发场景手动复现每一种锁 · 亲眼看到锁等待与死锁 · 含完整排查 SOP

---

## 目录

1. [环境准备：建表与造数据](#第一章环境准备建表与造数据)
2. [实战一：行锁基础——加锁、阻塞、释放](#第二章实战一行锁基础加锁阻塞释放)
3. [实战二：无索引的锁升级——行锁变表锁](#第三章实战二无索引的锁升级行锁变表锁)
4. [实战三：间隙锁——幻读防护与插入阻塞](#第四章实战三间隙锁幻读防护与插入阻塞)
5. [实战四：死锁复现与排查](#第五章实战四死锁复现与排查)
6. [实战五：MDL 元数据锁——ALTER TABLE 卡死复现](#第六章实战五mdl-元数据锁alter-table-卡死复现)
7. [实战六：库存超卖——三种方案对比](#第七章实战六库存超卖三种方案对比)
8. [实战七：转账死锁防护](#第八章实战七转账死锁防护)
9. [综合练习题](#第九章综合练习题)

---

## 第一章：环境准备：建表与造数据

> **目标**：搭建本章所有实战场景依赖的表结构和数据，后续所有实战都在这里跑。

### 1.1 使用的数据库

```sql
CREATE DATABASE IF NOT EXISTS lock_practice DEFAULT CHARACTER SET utf8mb4;
USE lock_practice;
```

### 1.2 用户表（复用 RBAC 场景）

```sql
-- 用户表：模拟企业用户系统
CREATE TABLE sys_user (
    id          BIGINT      NOT NULL AUTO_INCREMENT PRIMARY KEY,
    username    VARCHAR(50) NOT NULL COMMENT '用户名',
    dept_id     INT         NOT NULL COMMENT '部门 ID',
    status      TINYINT     NOT NULL DEFAULT 1 COMMENT '状态 1启用 0禁用',
    create_time DATETIME    NOT NULL DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 故意先不加索引，实战二中会用到"无索引→锁升级"的场景
```

### 1.3 商品库存表（超卖实战用）

```sql
-- 商品表：模拟电商库存
CREATE TABLE product (
    id          INT         NOT NULL AUTO_INCREMENT PRIMARY KEY,
    name        VARCHAR(100) NOT NULL COMMENT '商品名称',
    stock       INT         NOT NULL DEFAULT 0 COMMENT '库存数量',
    version     INT         NOT NULL DEFAULT 0 COMMENT '乐观锁版本号',
    price       DECIMAL(10,2) NOT NULL DEFAULT 0.00,
    update_time DATETIME    NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### 1.4 账户表（转账死锁实战用）

```sql
-- 账户表：模拟银行转账
CREATE TABLE account (
    id      BIGINT      NOT NULL AUTO_INCREMENT PRIMARY KEY,
    name    VARCHAR(50) NOT NULL COMMENT '账户名',
    balance DECIMAL(12,2) NOT NULL DEFAULT 0.00 COMMENT '余额'
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### 1.5 订单去重表（幂等实战用）

```sql
-- 订单幂等表：防重复提交
CREATE TABLE order_dedup (
    request_id  VARCHAR(64) NOT NULL PRIMARY KEY COMMENT '请求唯一 ID',
    order_id    BIGINT      NOT NULL,
    create_time DATETIME    NOT NULL DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### 1.6 造数据

```sql
-- ---- 插入用户数据（10 万条，dept_id 分布在 1~100）----
DELIMITER $$
CREATE PROCEDURE insert_users()
BEGIN
    DECLARE i INT DEFAULT 1;
    WHILE i <= 100000 DO
        INSERT INTO sys_user(username, dept_id, status, create_time)
        VALUES (
            CONCAT('user_', LPAD(i, 6, '0')),
            (i MOD 100) + 1,
            IF(i MOD 20 = 0, 0, 1),
            DATE_SUB(NOW(), INTERVAL FLOOR(RAND() * 365) DAY)
        );
        SET i = i + 1;
    END WHILE;
END$$
DELIMITER ;

CALL insert_users();
DROP PROCEDURE insert_users;

-- 验证
SELECT COUNT(*) FROM sys_user;  -- 应为 100000
```

```sql
-- ---- 插入商品数据（10 件商品）----
INSERT INTO product(name, stock, price) VALUES
    ('iPhone 15',      100, 6999.00),
    ('MacBook Pro',     50, 14999.00),
    ('AirPods Pro',    200, 1799.00),
    ('iPad Air',        80, 4799.00),
    ('Apple Watch',    150, 2999.00),
    ('小米 14',        120, 3999.00),
    ('华为 Mate60',    300, 6999.00),
    ('OPPO Find X7',  200, 5999.00),
    ('vivo X100',      80, 3999.00),
    ('荣耀 Magic6',   100, 3999.00);

-- 验证
SELECT * FROM product;
```

```sql
-- ---- 插入账户数据 ----
INSERT INTO account(name, balance) VALUES
    ('张三', 10000.00),
    ('李四', 10000.00),
    ('王五', 50000.00),
    ('赵六', 20000.00);

-- 验证
SELECT * FROM account;
```

### 1.7 环境检查

```sql
-- 确认隔离级别（默认应为 REPEATABLE-READ）
SELECT @@transaction_isolation;

-- 确认 InnoDB（所有表必须是 InnoDB）
SELECT TABLE_NAME, ENGINE FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'lock_practice';

-- 确认 performance_schema 已开启（监控锁等待用）
SHOW VARIABLES LIKE 'performance_schema';  -- 应为 ON
```

---

## 第二章：实战一：行锁基础——加锁、阻塞、释放

> **目标**：用两个会话手动复现行锁的加锁、阻塞等待、释放全过程，建立直觉。

**本章需要两个 MySQL 连接（会话 A、会话 B），并排打开。**

### 2.1 Step 1：先给 sys_user 加主键索引（确保走行锁）

```sql
-- 确认当前索引（此时应只有主键）
SHOW INDEX FROM sys_user;
```

### 2.2 Step 2：会话 A 加 X 锁，不提交

```sql
-- === 会话 A 执行 ===
BEGIN;
SELECT * FROM sys_user WHERE id = 1 FOR UPDATE;
-- 成功返回，id=1 的行现在被 X 锁锁住
-- 注意：此时不要 COMMIT，保持事务开着
```

### 2.3 Step 3：会话 B 尝试锁同一行——阻塞

```sql
-- === 会话 B 执行 ===
-- 尝试获取同一行的 X 锁
SELECT * FROM sys_user WHERE id = 1 FOR UPDATE;
-- 预期：光标转圈，阻塞等待，不返回结果
```

**此时去观察锁等待情况：**

```sql
-- === 另开一个会话（会话 C）执行监控 SQL ===

-- 查看当前锁等待
SELECT
    r.THREAD_ID          AS waiting_thread,
    r.OBJECT_NAME        AS locked_table,
    r.LOCK_MODE          AS waiting_for,
    b.THREAD_ID          AS blocking_thread,
    b.LOCK_MODE          AS blocking_with
FROM performance_schema.data_lock_waits w
JOIN performance_schema.data_locks r ON r.ENGINE_LOCK_ID = w.REQUESTING_ENGINE_LOCK_ID
JOIN performance_schema.data_locks b ON b.ENGINE_LOCK_ID = w.BLOCKING_ENGINE_LOCK_ID;
```

**预期输出：**

```
+----------------+--------------+-------------+-----------------+---------------+
| waiting_thread | locked_table | waiting_for | blocking_thread | blocking_with |
+----------------+--------------+-------------+-----------------+---------------+
|             XX | sys_user     | X,REC_NOT_GAP | YY            | X,REC_NOT_GAP |
+----------------+--------------+-------------+-----------------+---------------+
```

### 2.4 Step 4：会话 A 提交，观察会话 B 解除阻塞

```sql
-- === 会话 A 执行 ===
COMMIT;
-- 会话 B 立刻获得锁并返回结果
```

### 2.5 Step 5：验证不同行不互斥

```sql
-- === 会话 A 执行 ===
BEGIN;
SELECT * FROM sys_user WHERE id = 1 FOR UPDATE;  -- 锁住 id=1

-- === 会话 B 执行（同时）===
BEGIN;
SELECT * FROM sys_user WHERE id = 2 FOR UPDATE;  -- 锁住 id=2，立刻成功，不阻塞 ✅
COMMIT;

-- === 会话 A 执行 ===
COMMIT;
```

> ✅ **结论**：InnoDB 行锁锁的是索引记录，不同行互不干扰。

### 2.6 Step 6：验证 S 锁与 X 锁的兼容性

```sql
-- === 会话 A 执行 ===
BEGIN;
SELECT * FROM sys_user WHERE id = 1 FOR SHARE;  -- 加 S 锁（读锁）

-- === 会话 B 执行 ===
BEGIN;
SELECT * FROM sys_user WHERE id = 1 FOR SHARE;  -- 也加 S 锁，立刻成功 ✅（S+S 兼容）

SELECT * FROM sys_user WHERE id = 1 FOR UPDATE; -- 加 X 锁，阻塞 ❌（S+X 不兼容）
```

**查看当前所有行锁：**

```sql
-- 查看 data_locks 中的所有锁信息
SELECT
    OBJECT_NAME,
    INDEX_NAME,
    LOCK_TYPE,
    LOCK_MODE,
    LOCK_STATUS,
    LOCK_DATA
FROM performance_schema.data_locks
WHERE OBJECT_NAME = 'sys_user';
```

```sql
-- 清理：两个会话都 ROLLBACK
-- 会话 A：
ROLLBACK;
-- 会话 B：
ROLLBACK;
```

---

## 第三章：实战二：无索引的锁升级——行锁变表锁

> **目标**：亲眼验证 WHERE 条件没走索引时，InnoDB 会给每一行都加锁，等价于全表锁。

### 3.1 确认 sys_user 当前无 dept_id 索引

```sql
SHOW INDEX FROM sys_user;
-- 此时应只有 PRIMARY KEY，没有 dept_id 索引
```

### 3.2 Step 1：会话 A 用无索引列加锁

```sql
-- === 会话 A 执行 ===
BEGIN;
-- dept_id 没有索引，WHERE 走全表扫描
SELECT * FROM sys_user WHERE dept_id = 5 FOR UPDATE;
-- 执行成功，但 InnoDB 实际上锁了所有扫描过的行（全表！）
```

### 3.3 Step 2：会话 B 更新完全不同 dept_id 的行——也阻塞！

```sql
-- === 会话 B 执行 ===
-- 更新 dept_id=99 的用户（和 A 锁的 dept_id=5 完全不同）
UPDATE sys_user SET status = 0 WHERE id = 50000;
-- 预期：阻塞！因为 A 锁住了全表所有行

-- 更新一个 dept_id 完全不同的行也会阻塞
SELECT * FROM sys_user WHERE id = 50000 FOR UPDATE;
-- 预期：同样阻塞
```

**观察锁的数量（全表扫描时锁的行数极多）：**

```sql
-- === 会话 C ===
-- 统计当前持有的锁数量
SELECT COUNT(*) FROM performance_schema.data_locks
WHERE OBJECT_NAME = 'sys_user' AND LOCK_STATUS = 'GRANTED';
-- 预期：数量非常多（接近全表行数）
```

### 3.4 Step 3：会话 A ROLLBACK，然后加索引，再对比

```sql
-- === 会话 A 执行 ===
ROLLBACK;

-- 给 dept_id 加索引
ALTER TABLE sys_user ADD INDEX idx_dept_id(dept_id);
```

### 3.5 Step 4：有索引后再次测试

```sql
-- === 会话 A 执行 ===
BEGIN;
SELECT * FROM sys_user WHERE dept_id = 5 FOR UPDATE;
-- 现在走 idx_dept_id 索引，只锁 dept_id=5 的行

-- === 会话 B 执行 ===
-- 更新 dept_id=99 的行（不同 dept_id）
UPDATE sys_user SET status = 0 WHERE dept_id = 99;
-- 预期：立刻成功，不阻塞 ✅（行锁只锁 dept_id=5 的行）
```

```sql
-- 再次查看锁数量，明显减少
SELECT COUNT(*) FROM performance_schema.data_locks
WHERE OBJECT_NAME = 'sys_user' AND LOCK_STATUS = 'GRANTED';
-- 预期：只有 dept_id=5 对应的行数（约 1000 行）

-- 清理
-- 会话 A：
ROLLBACK;
```

> ⚠️ **结论**：生产环境 `FOR UPDATE` 或 `UPDATE/DELETE` 的 WHERE 列**必须有索引**，否则行锁退化为全表锁，并发彻底串行化。

---

## 第四章：实战三：间隙锁——幻读防护与插入阻塞

> **目标**：复现间隙锁的加锁范围，观察它如何阻止幻读，以及如何意外阻塞正常插入。

### 4.1 准备干净的测试数据

```sql
-- 用 product 表做实验（stock 列没有索引，先加一个普通索引）
ALTER TABLE product ADD INDEX idx_stock(stock);

-- 查看当前 stock 值分布（间隙锁的范围依赖索引值分布）
SELECT id, name, stock FROM product ORDER BY stock;
```

**预期数据（stock 列的索引值）：**

```
id=2  stock=50   (MacBook Pro)
id=4  stock=80   (iPad Air)
id=9  stock=80   (vivo X100)
id=1  stock=100  (iPhone 15)
id=10 stock=100  (荣耀)
id=6  stock=120  (小米14)
id=5  stock=150  (Apple Watch)
id=8  stock=200  (OPPO)
id=3  stock=200  (AirPods)
id=7  stock=300  (华为)
```

> 间隙锁区间（stock 列）：(-∞,50) (50,80) (80,100) (100,120) (120,150) (150,200) (200,300) (300,+∞)

### 4.2 Step 1：等值查询不存在的值——触发间隙锁

```sql
-- === 会话 A 执行 ===
BEGIN;
-- stock=90 不存在，触发间隙锁 (80, 100)
SELECT * FROM product WHERE stock = 90 FOR UPDATE;
-- 返回空结果，但间隙锁已加在 (80, 100) 区间
```

### 4.3 Step 2：尝试在间隙内插入——阻塞

```sql
-- === 会话 B 执行 ===
-- 插入 stock=85，在间隙 (80,100) 内
INSERT INTO product(name, stock, price) VALUES('测试商品A', 85, 999.00);
-- 预期：阻塞！被间隙锁拦住

-- 插入 stock=95，也在间隙 (80,100) 内
INSERT INTO product(name, stock, price) VALUES('测试商品B', 95, 999.00);
-- 预期：同样阻塞
```

### 4.4 Step 3：在间隙外插入——不阻塞

```sql
-- === 另开会话 C 执行 ===
-- 插入 stock=50，在 (50,80) 间隙之外（stock=50 本身存在，会加记录锁，换个值）
INSERT INTO product(name, stock, price) VALUES('测试商品C', 55, 999.00);
-- 预期：看情况（55 在间隙 (50,80) 内，也会被锁）

INSERT INTO product(name, stock, price) VALUES('测试商品D', 30, 999.00);
-- stock=30 在间隙 (-∞,50) 内，不被 A 的锁影响，成功 ✅
```

**查看当前间隙锁：**

```sql
-- === 会话 C ===
SELECT
    OBJECT_NAME,
    INDEX_NAME,
    LOCK_MODE,       -- X,GAP 表示间隙锁
    LOCK_STATUS,
    LOCK_DATA        -- 间隙锁的右边界值
FROM performance_schema.data_locks
WHERE OBJECT_NAME = 'product';
-- 找到 LOCK_MODE = 'X,GAP' 的记录，LOCK_DATA 就是间隙右边界
```

### 4.5 Step 4：范围查询的间隙锁范围

```sql
-- 先清理
-- 会话 A：ROLLBACK
-- 会话 B、C：ROLLBACK

-- === 会话 A 执行 ===
BEGIN;
-- 范围查询，锁住 stock 在 [100, 150] 的行及其间隙
SELECT * FROM product WHERE stock BETWEEN 100 AND 150 FOR UPDATE;
```

```sql
-- === 会话 B 执行 ===
-- 尝试插入 stock=110（在 100~150 范围内的间隙）
INSERT INTO product(name, stock, price) VALUES('测试商品E', 110, 999.00);
-- 预期：阻塞 ❌

-- 尝试插入 stock=160（刚好在 150~200 的间隙，也在临键锁范围内）
INSERT INTO product(name, stock, price) VALUES('测试商品F', 160, 999.00);
-- 预期：也阻塞！临键锁会多锁一个间隙

-- 尝试插入 stock=30（远在范围外）
INSERT INTO product(name, stock, price) VALUES('测试商品G', 30, 999.00);
-- 预期：成功 ✅
```

### 4.6 Step 5：READ COMMITTED 下无间隙锁

```sql
-- 清理所有事务
-- 会话 A：ROLLBACK

-- === 会话 A 用 RC 隔离级别重新测试 ===
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
BEGIN;
SELECT * FROM product WHERE stock = 90 FOR UPDATE;  -- 值不存在

-- === 会话 B ===
INSERT INTO product(name, stock, price) VALUES('测试商品H', 85, 999.00);
-- RC 下没有间隙锁，立刻成功 ✅

-- 清理
-- 会话 A：ROLLBACK
-- 会话 B：ROLLBACK（或 COMMIT）
DELETE FROM product WHERE name LIKE '测试商品%';

-- 恢复隔离级别
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
```

> ✅ **结论**：间隙锁是 REPEATABLE READ 级别的专属武器，RC 级别下不存在。生产上如果间隙锁导致并发问题，可以评估降到 RC。

---

## 第五章：实战四：死锁复现与排查

> **目标**：手动制造死锁，观察 MySQL 的自动检测和回滚，然后用 INNODB STATUS 排查。

### 5.1 实战一：经典两事务互等死锁

```sql
-- === 会话 A 执行（第 1 步）===
BEGIN;
UPDATE account SET balance = balance - 100 WHERE id = 1;  -- 锁住 id=1
-- 不要 COMMIT，等待下一步

-- === 会话 B 执行（第 2 步）===
BEGIN;
UPDATE account SET balance = balance - 100 WHERE id = 2;  -- 锁住 id=2
-- 不要 COMMIT，等待下一步

-- === 会话 A 执行（第 3 步）===
UPDATE account SET balance = balance + 100 WHERE id = 2;  -- 等待 B 释放 id=2

-- === 会话 B 执行（第 4 步，与 A 的第 3 步同时触发）===
UPDATE account SET balance = balance + 100 WHERE id = 1;  -- 等待 A 释放 id=1
-- → 死锁！MySQL 自动检测到环形等待，选择一个事务回滚
```

**预期现象：** 其中一个会话收到错误：

```
ERROR 1213 (40001): Deadlock found when trying to get lock;
try restarting transaction
```

### 5.2 Step 2：用 INNODB STATUS 排查死锁

```sql
-- 立即查看（死锁信息会被下一次死锁覆盖，要及时看）
SHOW ENGINE INNODB STATUS\G
```

**找到 `LATEST DETECTED DEADLOCK` 段，重点解读：**

```
------------------------
LATEST DETECTED DEADLOCK
------------------------
*** (1) TRANSACTION:
TRANSACTION XXXXX, ACTIVE X sec starting index read
...
*** (1) HOLDS THE LOCK(S):
RECORD LOCKS space id XX page no XX n bits XX
index PRIMARY of table `lock_practice`.`account`
trx id XXXXX lock_mode X locks rec but not gap
-- 事务1 持有的锁（id=1 的记录锁）

*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS ...
index PRIMARY of table `lock_practice`.`account`
trx id XXXXX lock_mode X locks rec but not gap
-- 事务1 在等待的锁（id=2 的记录锁）

*** (2) TRANSACTION:
...
*** (2) HOLDS THE LOCK(S):     ← 持有 id=2 的锁
*** (2) WAITING FOR THIS LOCK TO BE GRANTED:  ← 在等 id=1 的锁

*** WE ROLL BACK TRANSACTION (2)  ← MySQL 选代价小的事务回滚
```

### 5.3 开启死锁日志持久化

```sql
-- 将每次死锁都写入 MySQL 错误日志文件（生产环境建议开启）
SET GLOBAL innodb_print_all_deadlocks = ON;

-- 验证
SHOW VARIABLES LIKE 'innodb_print_all_deadlocks';
```

### 5.4 实战二：间隙锁 + 插入意向锁死锁（高频场景！）

```sql
-- product 表 stock 索引值：50, 80, 100, 120, 150, 200, 300
-- 间隙 (80, 100)

-- === 会话 A 执行（第 1 步）===
BEGIN;
-- stock=90 不存在，加间隙锁 (80, 100)
SELECT * FROM product WHERE stock = 90 FOR UPDATE;

-- === 会话 B 执行（第 2 步）===
BEGIN;
-- stock=91 不存在，也加间隙锁 (80, 100)（间隙锁之间不互斥，成功）
SELECT * FROM product WHERE stock = 91 FOR UPDATE;

-- === 会话 A 执行（第 3 步）===
-- 插入 stock=95（插入意向锁），被 B 的间隙锁阻塞
INSERT INTO product(name, stock, price) VALUES('新商品A', 95, 999.00);

-- === 会话 B 执行（第 4 步）===
-- 也插入 stock=93（插入意向锁），被 A 的间隙锁阻塞
INSERT INTO product(name, stock, price) VALUES('新商品B', 93, 999.00);
-- → 死锁！A 等 B 释放间隙锁，B 等 A 释放间隙锁
```

**排查并理解：**

```sql
SHOW ENGINE INNODB STATUS\G
-- 可以看到两个事务各自持有间隙锁，各自在等插入意向锁
-- LOCK_MODE: X,GAP（间隙锁）vs INSERT_INTENTION（插入意向锁）
```

### 5.5 设置死锁等待超时

```sql
-- 查看当前锁等待超时（默认 50 秒）
SHOW VARIABLES LIKE 'innodb_lock_wait_timeout';

-- 设置为 5 秒（推荐生产环境调小，让阻塞的事务快速失败而不是一直等）
SET innodb_lock_wait_timeout = 5;

-- 测试：会话 A 持有锁不释放，会话 B 等待超过 5 秒
-- 会话 B 会收到：ERROR 1205: Lock wait timeout exceeded
```

---

## 第六章：实战五：MDL 元数据锁——ALTER TABLE 卡死复现

> **目标**：复现线上最常见的 DDL 卡死场景，掌握排查和解决方法。

### 6.1 Step 1：会话 A 开启长事务（不提交）

```sql
-- === 会话 A 执行 ===
BEGIN;
SELECT * FROM sys_user WHERE id = 1;
-- 注意：这个普通 SELECT 会持有 MDL 读锁
-- 不要 COMMIT，模拟一个忘记提交的长事务
```

### 6.2 Step 2：DBA 执行 ALTER TABLE——被阻塞

```sql
-- === 会话 B（模拟 DBA）执行 ===
ALTER TABLE sys_user ADD COLUMN remark VARCHAR(200);
-- 预期：阻塞！一直转圈等待 MDL 写锁
-- ⚠️ 此时 sys_user 表的所有后续操作都会被排在 ALTER 后面排队
```

### 6.3 Step 3：会话 C 来了——也被阻塞

```sql
-- === 会话 C 执行（模拟正常业务请求）===
SELECT * FROM sys_user WHERE id = 2;
-- 预期：阻塞！被排在 ALTER TABLE 的 MDL 写锁申请后面
-- → 整张表"假死"，所有请求积压
```

### 6.4 Step 4：用 PROCESSLIST 和 metadata_locks 排查

```sql
-- === 会话 D（排查）===

-- 方法一：SHOW PROCESSLIST 快速看
SHOW FULL PROCESSLIST;
-- 找 State = "Waiting for table metadata lock" 的线程
-- 找 State = "Sleep" 且 Time 很大的线程（就是长事务）

-- 方法二：performance_schema 精确排查
SELECT
    ml.OBJECT_NAME         AS table_name,
    ml.LOCK_TYPE           AS lock_type,
    ml.LOCK_STATUS         AS lock_status,
    ml.OWNER_THREAD_ID     AS thread_id,
    ps.PROCESSLIST_USER    AS user_name,
    ps.PROCESSLIST_STATE   AS state,
    ps.PROCESSLIST_INFO    AS current_sql
FROM performance_schema.metadata_locks ml
JOIN performance_schema.threads ps ON ps.THREAD_ID = ml.OWNER_THREAD_ID
WHERE ml.OBJECT_NAME = 'sys_user'
ORDER BY ml.LOCK_STATUS;
-- LOCK_STATUS=GRANTED 且 LOCK_TYPE=SHARED_READ 的就是长事务（持有 MDL 读锁）
-- LOCK_STATUS=PENDING 且 LOCK_TYPE=EXCLUSIVE 的就是 ALTER TABLE（在等）
```

### 6.5 Step 5：KILL 长事务，解除阻塞

```sql
-- 根据上面查到的线程 ID（会话 A 的 thread_id）
-- KILL 会话 A 的连接（PROCESSLIST 里的 Id 列，不是 THREAD_ID）
KILL <会话A的连接Id>;

-- 会话 B 的 ALTER TABLE 立刻获得 MDL 写锁，继续执行
-- 会话 C 的 SELECT 也排在后面等 ALTER 完成后执行
```

### 6.6 生产建议：ALTER 加超时控制

```sql
-- MySQL 5.6+ 支持 WAIT N（超时秒数），超时放弃而非无限等待
ALTER TABLE sys_user
    WAIT 3
    DROP COLUMN remark;   -- 最多等 3 秒，超时返回错误，不阻塞业务

-- 清理（如果上面 ADD COLUMN 成功了）
ALTER TABLE sys_user DROP COLUMN IF EXISTS remark;
```

---

## 第七章：实战六：库存超卖——三种方案对比

> **目标**：用模拟并发的方式，直观对比三种防超卖方案的效果与性能差异。

### 7.1 准备：重置库存

```sql
-- 把 iPhone 15（id=1）的库存设为 10，模拟有限库存
UPDATE product SET stock = 10, version = 0 WHERE id = 1;
SELECT id, name, stock, version FROM product WHERE id = 1;
```

### 7.2 方案一：❌ 先查后改（错误示范，高并发超卖）

```sql
-- 模拟两个并发请求同时执行以下逻辑：
-- （真实并发需要两个会话同时执行，这里演示时序）

-- === 会话 A（请求1）===
BEGIN;
SELECT stock FROM product WHERE id = 1;   -- stock=10，够！
-- （此时会话 B 也查到 stock=10）

-- === 会话 B（请求2）===
BEGIN;
SELECT stock FROM product WHERE id = 1;   -- stock=10，够！

-- === 会话 A ===
UPDATE product SET stock = stock - 1 WHERE id = 1;  -- stock 变 9
COMMIT;

-- === 会话 B ===
UPDATE product SET stock = stock - 1 WHERE id = 1;  -- stock 变 8（应该是 9！）
COMMIT;
-- 两个请求都成功，但实际只消耗了 1 份库存，却扣了 2 次 → 超卖
```

> ❌ **问题根源**：普通 SELECT 走 MVCC 快照读，不加锁，两个事务读到的都是旧值。

### 7.3 方案二：✅ WHERE 加条件（推荐，原子操作）

```sql
-- 重置库存
UPDATE product SET stock = 10 WHERE id = 1;

-- 正确写法：把检查和扣减合并在一条 UPDATE 里，利用 UPDATE 的原子性
-- 模拟并发：两个会话同时执行
-- === 会话 A ===
UPDATE product SET stock = stock - 1 WHERE id = 1 AND stock > 0;
SELECT ROW_COUNT();  -- 1 = 扣减成功, 0 = 库存不足

-- === 会话 B（同时）===
UPDATE product SET stock = stock - 1 WHERE id = 1 AND stock > 0;
SELECT ROW_COUNT();  -- 1 = 扣减成功, 0 = 库存不足
```

**连续执行 10 次，验证库存不会变为负数：**

```sql
-- 重置
UPDATE product SET stock = 3 WHERE id = 1;

-- 模拟 5 次请求（超过库存量）
UPDATE product SET stock = stock - 1 WHERE id = 1 AND stock > 0; SELECT ROW_COUNT() AS result;
UPDATE product SET stock = stock - 1 WHERE id = 1 AND stock > 0; SELECT ROW_COUNT() AS result;
UPDATE product SET stock = stock - 1 WHERE id = 1 AND stock > 0; SELECT ROW_COUNT() AS result;
UPDATE product SET stock = stock - 1 WHERE id = 1 AND stock > 0; SELECT ROW_COUNT() AS result; -- 库存=0，ROW_COUNT=1
UPDATE product SET stock = stock - 1 WHERE id = 1 AND stock > 0; SELECT ROW_COUNT() AS result; -- 库存不足，ROW_COUNT=0 ✅

-- 验证最终库存
SELECT id, name, stock FROM product WHERE id = 1;
-- stock 应为 0，不会出现负数
```

### 7.4 方案三：✅ SELECT FOR UPDATE（悲观锁，需要业务判断时用）

```sql
-- 重置
UPDATE product SET stock = 10 WHERE id = 1;

-- === 会话 A ===
BEGIN;
SELECT stock FROM product WHERE id = 1 FOR UPDATE;   -- 加 X 锁，读最新值
-- 应用层判断 stock > 0
UPDATE product SET stock = stock - 1 WHERE id = 1;
COMMIT;

-- === 会话 B（与 A 并发）===
BEGIN;
SELECT stock FROM product WHERE id = 1 FOR UPDATE;   -- 等待 A 释放 X 锁
-- A COMMIT 后，B 获得锁，读到最新 stock=9
UPDATE product SET stock = stock - 1 WHERE id = 1;   -- stock 变 8
COMMIT;
-- ✅ 串行执行，绝对安全，但并发吞吐降低
```

**对比两种方案：**

```sql
-- 查看 FOR UPDATE 的锁情况（会话 A 持有锁时查）
SELECT LOCK_MODE, LOCK_STATUS, LOCK_DATA
FROM performance_schema.data_locks
WHERE OBJECT_NAME = 'product';
-- 看到 PRIMARY 上的 X,REC_NOT_GAP 锁
```

### 7.5 方案四：✅ 乐观锁（version 号，低竞争场景）

```sql
-- 重置
UPDATE product SET stock = 10, version = 0 WHERE id = 1;

-- 模拟乐观锁流程：先读版本号，更新时带版本号校验
-- === 请求1 ===
-- Step 1：读出当前版本号
SELECT stock, version FROM product WHERE id = 1;   -- stock=10, version=0

-- Step 2：带版本号更新（CAS 操作）
UPDATE product
SET stock = stock - 1, version = version + 1
WHERE id = 1 AND stock > 0 AND version = 0;        -- 版本号匹配才更新
SELECT ROW_COUNT();  -- 1 = 成功

-- === 请求2（同时读到 version=0，但请求1已经改成 version=1）===
UPDATE product
SET stock = stock - 1, version = version + 1
WHERE id = 1 AND stock > 0 AND version = 0;        -- version=0 已过期！
SELECT ROW_COUNT();  -- 0 = 更新失败（version 不匹配）→ 应用层重试

-- 重试：重新读版本号
SELECT stock, version FROM product WHERE id = 1;   -- stock=9, version=1
UPDATE product
SET stock = stock - 1, version = version + 1
WHERE id = 1 AND stock > 0 AND version = 1;        -- 成功
```

**三种方案对比总结：**

| 方案 | 并发安全 | 实现复杂度 | 并发吞吐 | 适用场景 |
|------|---------|-----------|---------|---------|
| 先查后改 | ❌ 超卖 | 低 | 高 | 不适用 |
| WHERE 加条件 | ✅ | 低 | 高 | **大多数场景首选** |
| SELECT FOR UPDATE | ✅ | 中 | 低 | 需要业务判断时 |
| 乐观锁 version | ✅（低竞争） | 中 | 中 | 低并发、允许重试 |

---

## 第八章：实战七：转账死锁防护

> **目标**：复现转账场景的死锁，对比正确写法，理解"固定加锁顺序"的本质。

### 8.1 准备数据

```sql
-- 查看账户余额
SELECT * FROM account;
-- id=1 张三 10000, id=2 李四 10000, id=3 王五 50000, id=4 赵六 20000
```

### 8.2 Step 1：错误写法——复现死锁

```sql
-- 场景：张三（id=1）→ 李四（id=2）转 100 元
--       李四（id=2）→ 张三（id=1）转 200 元（同时发生）

-- === 会话 A：张三→李四 ===
BEGIN;
UPDATE account SET balance = balance - 100 WHERE id = 1;  -- Step1: 锁 id=1
-- （暂停，等会话 B 执行 Step1）

-- === 会话 B：李四→张三 ===
BEGIN;
UPDATE account SET balance = balance - 200 WHERE id = 2;  -- Step1: 锁 id=2
-- （暂停，等会话 A 执行 Step2）

-- === 会话 A：Step2 ===
UPDATE account SET balance = balance + 100 WHERE id = 2;  -- 等待 B 释放 id=2

-- === 会话 B：Step2 ===
UPDATE account SET balance = balance + 200 WHERE id = 1;  -- 等待 A 释放 id=1
-- → 死锁！
```

**马上查看死锁详情：**

```sql
SHOW ENGINE INNODB STATUS\G
-- 找 LATEST DETECTED DEADLOCK，记录下两个事务的 SQL 和锁信息
```

### 8.3 Step 2：重置数据

```sql
-- 死锁发生后，被回滚的事务数据已恢复，手动重置余额
UPDATE account SET balance = 10000 WHERE id IN (1, 2);
SELECT id, name, balance FROM account WHERE id IN (1, 2);
```

### 8.4 Step 3：正确写法——固定加锁顺序

```sql
-- 规则：总是先锁 id 较小的账户，再锁 id 较大的账户
-- 张三→李四：id=1 < id=2，先锁1再锁2
-- 李四→张三：id=2 > id=1，也必须先锁1再锁2（固定顺序！）

-- === 会话 A：张三→李四 ===
BEGIN;
-- 先锁 LEAST(1,2)=1，再锁 GREATEST(1,2)=2
SELECT * FROM account WHERE id = 1 FOR UPDATE;  -- 先锁 id=1
SELECT * FROM account WHERE id = 2 FOR UPDATE;  -- 再锁 id=2（顺序固定）
UPDATE account SET balance = balance - 100 WHERE id = 1;
UPDATE account SET balance = balance + 100 WHERE id = 2;
COMMIT;

-- === 会话 B：李四→张三（同时执行）===
BEGIN;
-- 虽然转账方向是 id=2→id=1，但加锁顺序也必须先1后2
SELECT * FROM account WHERE id = 1 FOR UPDATE;  -- 先等 A 释放 id=1 的锁
SELECT * FROM account WHERE id = 2 FOR UPDATE;  -- 再锁 id=2
UPDATE account SET balance = balance - 200 WHERE id = 2;
UPDATE account SET balance = balance + 200 WHERE id = 1;
COMMIT;
-- ✅ 不会死锁！B 等 A 完成后顺序执行，无循环等待
```

### 8.5 Step 4：更简洁——用 IN 批量锁（InnoDB 自动按主键顺序加锁）

```sql
-- InnoDB 对 IN 中的主键，会按照主键大小顺序加锁（自动有序！）
-- === 会话 A：张三→李四 ===
BEGIN;
SELECT * FROM account WHERE id IN (1, 2) FOR UPDATE;
-- InnoDB 自动按 id 升序加锁：先锁 id=1，再锁 id=2
UPDATE account SET balance = balance - 100 WHERE id = 1;
UPDATE account SET balance = balance + 100 WHERE id = 2;
COMMIT;

-- === 会话 B：李四→张三（同时）===
BEGIN;
SELECT * FROM account WHERE id IN (1, 2) FOR UPDATE;
-- InnoDB 同样自动按 id 升序加锁：先等 id=1，再锁 id=2
-- ✅ 不会死锁！
UPDATE account SET balance = balance - 200 WHERE id = 2;
UPDATE account SET balance = balance + 200 WHERE id = 1;
COMMIT;
```

**验证最终余额（两次转账都应该成功）：**

```sql
SELECT id, name, balance FROM account WHERE id IN (1, 2);
-- 张三：10000 - 100 + 200 = 10100
-- 李四：10000 + 100 - 200 = 9900
```

---

## 第九章：综合练习题

> 每道题先自己分析，再看答案。

---

### 练习 1：判断是否阻塞

**场景**：sys_user 表有 `idx_dept_id(dept_id)` 索引，当前数据中 dept_id 有值：1, 5, 10, 20, 50。

**会话 A 执行（未提交）：**
```sql
BEGIN;
SELECT * FROM sys_user WHERE dept_id = 10 FOR UPDATE;
```

**问以下操作哪个会阻塞？**

```sql
-- ① 会话 B：
UPDATE sys_user SET status = 0 WHERE dept_id = 10;

-- ② 会话 B：
UPDATE sys_user SET status = 0 WHERE dept_id = 20;

-- ③ 会话 B：
INSERT INTO sys_user(username, dept_id, status) VALUES('new_user', 12, 1);

-- ④ 会话 B：
INSERT INTO sys_user(username, dept_id, status) VALUES('new_user', 3, 1);

-- ⑤ 会话 B：
SELECT * FROM sys_user WHERE dept_id = 10;  -- 普通 SELECT（不加 FOR UPDATE）
```

<details>
<summary>点击查看答案</summary>

**① 阻塞** ❌：`dept_id=10` 有记录，会加记录锁 + 临键锁，UPDATE 需要 X 锁，冲突。

**② 不阻塞** ✅：`dept_id=20` 是另一个记录，临键锁不覆盖 `(10, 20]` 之外，UPDATE dept_id=20 走自己的行锁。

**③ 阻塞** ❌：`dept_id=12` 落在间隙 `(10, 20)` 内，被临键锁的间隙部分拦住。

**④ 不阻塞** ✅：`dept_id=3` 落在间隙 `(1, 5)`，不在会话 A 的锁范围内。

**⑤ 不阻塞** ✅：普通 SELECT 走 MVCC 快照读，不加锁，不受 FOR UPDATE 影响。

</details>

---

### 练习 2：找出死锁原因并修复

以下代码在高并发下频繁出现死锁，找出原因并给出修复方案：

```sql
-- 业务逻辑：给用户批量扣分（伪代码顺序）
-- 用户 A 同时对 user_id=1 和 user_id=3 扣分
-- 用户 B 同时对 user_id=3 和 user_id=1 扣分

-- 请求 A 的 SQL 顺序：
BEGIN;
UPDATE sys_user SET status = 0 WHERE id = 1;
UPDATE sys_user SET status = 0 WHERE id = 3;
COMMIT;

-- 请求 B 的 SQL 顺序：
BEGIN;
UPDATE sys_user SET status = 0 WHERE id = 3;  -- 顺序不同！
UPDATE sys_user SET status = 0 WHERE id = 1;
COMMIT;
```

<details>
<summary>点击查看答案</summary>

**死锁原因**：A 和 B 对同一批行的加锁顺序相反，形成循环等待：
- A 持有 id=1 的锁，等待 id=3
- B 持有 id=3 的锁，等待 id=1

**修复方案**：固定加锁顺序，始终按 id 升序更新：

```sql
-- 请求 A 和 请求 B 都改为按 id 升序操作
BEGIN;
UPDATE sys_user SET status = 0 WHERE id = 1;  -- id 小的先执行
UPDATE sys_user SET status = 0 WHERE id = 3;  -- id 大的后执行
COMMIT;

-- 或者一次性锁住所有行（InnoDB 按主键顺序自动加锁）
BEGIN;
SELECT * FROM sys_user WHERE id IN (1, 3) FOR UPDATE;
UPDATE sys_user SET status = 0 WHERE id IN (1, 3);
COMMIT;
```

</details>

---

### 练习 3：分析锁升级场景

**场景**：sys_user 目前只有主键索引，没有 status 索引。

```sql
-- 会话 A：
BEGIN;
UPDATE sys_user SET dept_id = 999 WHERE status = 0;
-- （status 没有索引，全表扫描）
```

**问：**
1. 会话 A 锁住了多少行？
2. 此时会话 B 执行 `UPDATE sys_user SET dept_id = 1 WHERE id = 50000` 会发生什么？
3. 如何解决？

<details>
<summary>点击查看答案</summary>

**1.** 锁住了**全表所有行**（约 10 万行）。status 没有索引，InnoDB 全表扫描时，每扫到一行都加 X 锁。

**2.** 会话 B **阻塞**。虽然 id=50000 是主键精准查询，但它对应的行被会话 A 的全表扫描锁住了，B 必须等 A 提交才能拿到该行的锁。

**3.** 解决方案：
```sql
-- 给 status 加索引，让 UPDATE 走索引，只锁 status=0 的行
ALTER TABLE sys_user ADD INDEX idx_status(status);
-- 加索引后，只有 status=0 的行会被锁，status=1 的行不受影响
```

</details>

---

### 练习 4：设计一个安全的抢券逻辑

**需求**：优惠券表如下，设计 SQL 保证高并发下：
1. 同一用户不能重复领券
2. 库存不能超卖
3. 尽量减少锁持有时间

```sql
CREATE TABLE coupon (
    id          BIGINT NOT NULL AUTO_INCREMENT PRIMARY KEY,
    name        VARCHAR(100) NOT NULL,
    total       INT    NOT NULL DEFAULT 0 COMMENT '总库存',
    claimed     INT    NOT NULL DEFAULT 0 COMMENT '已领取数量'
);

CREATE TABLE user_coupon (
    id          BIGINT NOT NULL AUTO_INCREMENT PRIMARY KEY,
    user_id     BIGINT NOT NULL,
    coupon_id   BIGINT NOT NULL,
    claim_time  DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY uk_user_coupon(user_id, coupon_id)  -- 防重复领
);
```

<details>
<summary>点击查看答案</summary>

```sql
-- 方案：利用唯一索引防重 + WHERE 条件防超卖，两步原子操作

-- Step 1：扣库存（WHERE 条件保证不超卖，无需显式 FOR UPDATE）
UPDATE coupon
SET claimed = claimed + 1
WHERE id = #{couponId}
  AND claimed < total;     -- 库存充足才扣减
-- affected rows = 0 → 库存不足，直接返回失败

-- Step 2：写领券记录（唯一索引防重复领）
INSERT INTO user_coupon(user_id, coupon_id)
VALUES (#{userId}, #{couponId});
-- Duplicate key（错误码 1062）→ 已领过，返回"已领取"

-- 优点：
-- 1. UPDATE 原子操作，无需 FOR UPDATE，行锁持有时间极短
-- 2. 唯一索引在数据库层面保证不重复，无需应用层加锁
-- 3. 两步都不需要显式事务包裹（各自原子）
-- 4. 适合高并发抢券场景
```

</details>

---

## 附录：本章常用诊断 SQL

```sql
-- 1. 查看当前锁等待
SELECT r.THREAD_ID waiting_thread, r.OBJECT_NAME table_name,
       r.LOCK_MODE waiting_mode, b.THREAD_ID blocking_thread, b.LOCK_MODE blocking_mode
FROM performance_schema.data_lock_waits w
JOIN performance_schema.data_locks r ON r.ENGINE_LOCK_ID = w.REQUESTING_ENGINE_LOCK_ID
JOIN performance_schema.data_locks b ON b.ENGINE_LOCK_ID = w.BLOCKING_ENGINE_LOCK_ID;

-- 2. 查看所有持有的锁
SELECT OBJECT_NAME, INDEX_NAME, LOCK_TYPE, LOCK_MODE, LOCK_STATUS, LOCK_DATA
FROM performance_schema.data_locks;

-- 3. 查看活跃事务
SELECT trx_id, trx_state, trx_started, trx_mysql_thread_id, trx_rows_locked, trx_query
FROM information_schema.innodb_trx ORDER BY trx_started;

-- 4. 查看 MDL 锁
SELECT OBJECT_NAME, LOCK_TYPE, LOCK_STATUS, OWNER_THREAD_ID
FROM performance_schema.metadata_locks
WHERE OBJECT_SCHEMA = 'lock_practice';

-- 5. 查看死锁详情
SHOW ENGINE INNODB STATUS\G

-- 6. 行锁统计
SHOW STATUS LIKE 'innodb_row_lock%';

-- 7. KILL 阻塞连接
-- KILL <PROCESSLIST 中的 Id>;
```
