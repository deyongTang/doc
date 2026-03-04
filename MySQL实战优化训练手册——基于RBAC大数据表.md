# MySQL 实战优化训练手册 —— 基于 RBAC 大数据表

> 用真实业务场景造百万级数据，手把手练习索引设计、慢 SQL 排查、执行计划分析

---

## 目录

1. [RBAC 业务场景设计](#第一章rbac-业务场景设计)
2. [建表：完整 DDL](#第二章建表完整-ddl)
3. [造数据：百万级测试数据生成](#第三章造数据百万级测试数据生成)
4. [实战一：全表扫描 → 补索引](#第四章实战一全表扫描--补索引)
5. [实战二：联合索引设计与最左前缀](#第五章实战二联合索引设计与最左前缀)
6. [实战三：覆盖索引消除回表](#第六章实战三覆盖索引消除回表)
7. [实战四：深分页优化](#第七章实战四深分页优化)
8. [实战五：JOIN 多表关联优化](#第八章实战五join-多表关联优化)
9. [实战六：ORDER BY / GROUP BY 优化](#第九章实战六order-by--group-by-优化)
10. [实战七：索引失效场景排查](#第十章实战七索引失效场景排查)
11. [综合练习题](#第十一章综合练习题)

---

## 第一章：RBAC 业务场景设计

### 1.1 什么是 RBAC

RBAC（Role-Based Access Control，基于角色的访问控制）是企业权限系统的标准模型。

**核心关系：**

```
用户(User) ──── 分配角色 ────▶ 角色(Role) ──── 包含权限 ────▶ 权限(Permission)
                 (多对多)                         (多对多)

用户 ──── 属于部门 ────▶ 部门(Dept)
```

### 1.2 数据规模设计（模拟中型企业）

| 表名 | 业务含义 | 数据量 | 备注 |
|------|---------|-------|------|
| `sys_user` | 用户表 | **100 万** | 核心大表 |
| `sys_role` | 角色表 | 100 条 | 小字典表 |
| `sys_permission` | 权限表 | 500 条 | 小字典表 |
| `sys_dept` | 部门表 | 1000 条 | 树形结构 |
| `sys_user_role` | 用户-角色关联 | **300 万** | 多对多，大表 |
| `sys_role_permission` | 角色-权限关联 | 5000 条 | 相对较小 |
| `sys_user_login_log` | 用户登录日志 | **500 万** | 最大表 |

> 这个规模在生产中属于「中小型系统」，足以让大部分索引问题完整复现。

---

## 第二章：建表：完整 DDL

> 建议在本地 MySQL 8.0 执行，先建库再建表。

```sql
CREATE DATABASE IF NOT EXISTS rbac_practice
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;

USE rbac_practice;
```

### 2.1 部门表（sys_dept）

```sql
CREATE TABLE sys_dept (
  id          INT UNSIGNED    NOT NULL AUTO_INCREMENT COMMENT '部门ID',
  parent_id   INT UNSIGNED    NOT NULL DEFAULT 0      COMMENT '父部门ID，0=根节点',
  dept_name   VARCHAR(50)     NOT NULL                COMMENT '部门名称',
  dept_code   VARCHAR(30)     NOT NULL                COMMENT '部门编码',
  sort        SMALLINT        NOT NULL DEFAULT 0      COMMENT '排序号',
  leader      VARCHAR(30)     NULL                    COMMENT '负责人姓名',
  status      TINYINT         NOT NULL DEFAULT 1      COMMENT '状态 1正常 0停用',
  create_time DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP,
  update_time DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  UNIQUE KEY uk_dept_code (dept_code),
  KEY idx_parent_id (parent_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='部门表';
```

### 2.2 角色表（sys_role）

```sql
CREATE TABLE sys_role (
  id          INT UNSIGNED    NOT NULL AUTO_INCREMENT COMMENT '角色ID',
  role_name   VARCHAR(50)     NOT NULL                COMMENT '角色名称',
  role_code   VARCHAR(50)     NOT NULL                COMMENT '角色编码',
  role_type   TINYINT         NOT NULL DEFAULT 1      COMMENT '角色类型 1系统角色 2自定义角色',
  status      TINYINT         NOT NULL DEFAULT 1      COMMENT '状态 1正常 0停用',
  remark      VARCHAR(200)    NULL                    COMMENT '备注',
  create_time DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP,
  update_time DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  UNIQUE KEY uk_role_code (role_code)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='角色表';
```

### 2.3 权限表（sys_permission）

```sql
CREATE TABLE sys_permission (
  id              INT UNSIGNED    NOT NULL AUTO_INCREMENT COMMENT '权限ID',
  parent_id       INT UNSIGNED    NOT NULL DEFAULT 0      COMMENT '父权限ID',
  perm_name       VARCHAR(50)     NOT NULL                COMMENT '权限名称',
  perm_code       VARCHAR(100)    NOT NULL                COMMENT '权限编码，如 user:list',
  perm_type       TINYINT         NOT NULL DEFAULT 1      COMMENT '类型 1菜单 2按钮 3接口',
  path            VARCHAR(200)    NULL                    COMMENT '路由路径',
  icon            VARCHAR(50)     NULL                    COMMENT '图标',
  sort            SMALLINT        NOT NULL DEFAULT 0,
  status          TINYINT         NOT NULL DEFAULT 1,
  create_time     DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  UNIQUE KEY uk_perm_code (perm_code),
  KEY idx_parent_id (parent_id),
  KEY idx_perm_type (perm_type)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='权限表';
```

### 2.4 用户表（sys_user）—— 核心大表

```sql
CREATE TABLE sys_user (
  id            BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '用户ID',
  username      VARCHAR(50)     NOT NULL                COMMENT '用户名（登录名）',
  password      VARCHAR(100)    NOT NULL                COMMENT '密码（BCrypt）',
  real_name     VARCHAR(30)     NOT NULL                COMMENT '真实姓名',
  mobile        VARCHAR(20)     NULL                    COMMENT '手机号',
  email         VARCHAR(100)    NULL                    COMMENT '邮箱',
  avatar        VARCHAR(200)    NULL                    COMMENT '头像URL',
  gender        TINYINT         NOT NULL DEFAULT 0      COMMENT '性别 0未知 1男 2女',
  dept_id       INT UNSIGNED    NOT NULL DEFAULT 0      COMMENT '所属部门ID',
  status        TINYINT         NOT NULL DEFAULT 1      COMMENT '状态 1正常 0禁用 2锁定',
  last_login_ip VARCHAR(50)     NULL                    COMMENT '最后登录IP',
  last_login_time DATETIME      NULL                    COMMENT '最后登录时间',
  login_count   INT             NOT NULL DEFAULT 0      COMMENT '登录次数',
  create_time   DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP,
  update_time   DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (id)
  -- ⚠️ 故意不建普通索引，后续实战中逐步加索引
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='用户表（100万条）';
```

### 2.5 用户-角色关联表（sys_user_role）

```sql
CREATE TABLE sys_user_role (
  id          BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  user_id     BIGINT UNSIGNED NOT NULL COMMENT '用户ID',
  role_id     INT UNSIGNED    NOT NULL COMMENT '角色ID',
  create_time DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (id)
  -- ⚠️ 故意不建索引
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='用户角色关联表（300万条）';
```

### 2.6 角色-权限关联表（sys_role_permission）

```sql
CREATE TABLE sys_role_permission (
  id            BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  role_id       INT UNSIGNED    NOT NULL COMMENT '角色ID',
  permission_id INT UNSIGNED    NOT NULL COMMENT '权限ID',
  create_time   DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  UNIQUE KEY uk_role_perm (role_id, permission_id),
  KEY idx_permission_id (permission_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='角色权限关联表';
```

### 2.7 用户登录日志表（sys_user_login_log）—— 最大表

```sql
CREATE TABLE sys_user_login_log (
  id          BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  user_id     BIGINT UNSIGNED NOT NULL                COMMENT '用户ID',
  username    VARCHAR(50)     NOT NULL                COMMENT '用户名（冗余，避免JOIN）',
  login_ip    VARCHAR(50)     NOT NULL                COMMENT '登录IP',
  login_time  DATETIME        NOT NULL                COMMENT '登录时间',
  logout_time DATETIME        NULL                    COMMENT '登出时间',
  device_type TINYINT         NOT NULL DEFAULT 1      COMMENT '设备类型 1PC 2APP 3小程序',
  login_result TINYINT        NOT NULL DEFAULT 1      COMMENT '结果 1成功 0失败',
  fail_reason VARCHAR(100)    NULL                    COMMENT '失败原因',
  PRIMARY KEY (id)
  -- ⚠️ 故意不建索引
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='登录日志表（500万条）';
```

---

## 第三章：造数据：百万级测试数据生成

### 3.1 插入基础字典数据

```sql
-- ---- 插入部门数据（1000条）----
DELIMITER $$
CREATE PROCEDURE insert_dept()
BEGIN
  DECLARE i INT DEFAULT 1;
  -- 先插入一级部门（10个）
  INSERT INTO sys_dept(parent_id, dept_name, dept_code, sort, leader, status)
  VALUES
    (0, '总裁办',     'CEO',    1,  '王总', 1),
    (0, '技术中心',   'TECH',   2,  '张总', 1),
    (0, '产品中心',   'PROD',   3,  '李总', 1),
    (0, '销售中心',   'SALES',  4,  '赵总', 1),
    (0, '市场中心',   'MKT',    5,  '孙总', 1),
    (0, '财务中心',   'FIN',    6,  '周总', 1),
    (0, '人力资源',   'HR',     7,  '吴总', 1),
    (0, '运营中心',   'OPS',    8,  '郑总', 1),
    (0, '客服中心',   'CS',     9,  '王二', 1),
    (0, '法务合规',   'LEGAL',  10, '冯总', 1);

  -- 插入二级部门（990个，挂在10个一级部门下）
  WHILE i <= 990 DO
    INSERT INTO sys_dept(parent_id, dept_name, dept_code, sort, status)
    VALUES (
      (i MOD 10) + 1,
      CONCAT('部门_', i),
      CONCAT('DEPT_', LPAD(i, 4, '0')),
      i MOD 100,
      IF(i MOD 10 = 0, 0, 1)
    );
    SET i = i + 1;
  END WHILE;
END$$
DELIMITER ;

CALL insert_dept();
DROP PROCEDURE insert_dept;
```

```sql
-- ---- 插入角色数据（100条）----
DELIMITER $$
CREATE PROCEDURE insert_roles()
BEGIN
  DECLARE i INT DEFAULT 1;
  INSERT INTO sys_role(role_name, role_code, role_type, status)
  VALUES ('超级管理员', 'SUPER_ADMIN', 1, 1),
         ('系统管理员', 'SYS_ADMIN',   1, 1),
         ('普通员工',   'EMPLOYEE',    1, 1),
         ('部门经理',   'DEPT_MANAGER',1, 1),
         ('只读用户',   'READ_ONLY',   1, 1);

  WHILE i <= 95 DO
    INSERT INTO sys_role(role_name, role_code, role_type, status)
    VALUES (
      CONCAT('自定义角色_', i),
      CONCAT('CUSTOM_ROLE_', LPAD(i, 3, '0')),
      2,
      IF(i MOD 10 = 0, 0, 1)
    );
    SET i = i + 1;
  END WHILE;
END$$
DELIMITER ;

CALL insert_roles();
DROP PROCEDURE insert_roles;
```

```sql
-- ---- 插入权限数据（500条）----
DELIMITER $$
CREATE PROCEDURE insert_permissions()
BEGIN
  DECLARE i INT DEFAULT 1;
  DECLARE modules CURSOR FOR
    SELECT 'user' UNION SELECT 'role' UNION SELECT 'dept'
    UNION SELECT 'perm' UNION SELECT 'log' UNION SELECT 'config'
    UNION SELECT 'report' UNION SELECT 'dashboard' UNION SELECT 'audit' UNION SELECT 'monitor';

  -- 插入一批标准权限
  INSERT INTO sys_permission(parent_id, perm_name, perm_code, perm_type)
  VALUES
    (0, '系统管理', 'system', 1),
    (0, '用户管理', 'system:user', 1),
    (0, '角色管理', 'system:role', 1),
    (0, '权限管理', 'system:perm', 1),
    (0, '日志管理', 'system:log',  1);

  WHILE i <= 495 DO
    INSERT INTO sys_permission(parent_id, perm_name, perm_code, perm_type)
    VALUES (
      (i MOD 5) + 1,
      CONCAT('权限_', i),
      CONCAT('module', (i MOD 10) + 1, ':action', i),
      (i MOD 3) + 1
    );
    SET i = i + 1;
  END WHILE;
END$$
DELIMITER ;

CALL insert_permissions();
DROP PROCEDURE insert_permissions;
```

### 3.2 生成 100 万用户数据

> ⏱️ 预计耗时：5~15 分钟（取决于机器性能）。建议分批插入，每批 1 万条。

```sql
DELIMITER $$
CREATE PROCEDURE insert_users(IN batch_start INT, IN batch_end INT)
BEGIN
  DECLARE i INT DEFAULT batch_start;

  -- 关闭 autocommit，批量提交提升性能
  SET autocommit = 0;

  WHILE i <= batch_end DO
    INSERT INTO sys_user(
      username, password, real_name, mobile, email,
      gender, dept_id, status, last_login_time, login_count, create_time
    ) VALUES (
      CONCAT('user_', LPAD(i, 7, '0')),
      '$2a$10$N9qo8uLOickgx2ZMRZoMyeIjZAgcfl7p92ldGxad68LJZdL17lhWy', -- BCrypt: 'password'
      CONCAT(
        ELT(1 + FLOOR(RAND() * 20), '赵','钱','孙','李','周','吴','郑','王','冯','陈',
            '褚','卫','蒋','沈','韩','杨','朱','秦','尤','许'),
        ELT(1 + FLOOR(RAND() * 20), '伟','芳','娜','秀英','敏','静','丽','强','磊','军',
            '洋','勇','艳','杰','娟','涛','明','超','秀兰','霞')
      ),
      CONCAT('1', ELT(1 + FLOOR(RAND() * 7), '30','31','32','33','50','51','88'),
             LPAD(FLOOR(RAND() * 100000000), 8, '0')),
      CONCAT('user_', i, '@example.com'),
      1 + FLOOR(RAND() * 2),
      1 + FLOOR(RAND() * 1000),          -- dept_id 1~1000
      ELT(1 + FLOOR(RAND() * 10), 1,1,1,1,1,1,1,1,0,2),  -- 80% 正常, 10% 禁用, 10% 锁定
      DATE_SUB(NOW(), INTERVAL FLOOR(RAND() * 365) DAY),
      FLOOR(RAND() * 500),
      DATE_SUB(NOW(), INTERVAL (1000001 - i) SECOND)
    );

    IF i MOD 1000 = 0 THEN
      COMMIT;  -- 每 1000 条提交一次
    END IF;

    SET i = i + 1;
  END WHILE;

  COMMIT;
END$$
DELIMITER ;

-- 分 100 批执行，每批 1 万条，避免单次事务过大
-- 可以开多个会话并行执行不同范围
CALL insert_users(1, 100000);
CALL insert_users(100001, 200000);
CALL insert_users(200001, 300000);
CALL insert_users(300001, 400000);
CALL insert_users(400001, 500000);
CALL insert_users(500001, 600000);
CALL insert_users(600001, 700000);
CALL insert_users(700001, 800000);
CALL insert_users(800001, 900000);
CALL insert_users(900001, 1000000);

DROP PROCEDURE insert_users;
```

### 3.3 生成 300 万用户-角色关联

```sql
DELIMITER $$
CREATE PROCEDURE insert_user_roles()
BEGIN
  DECLARE i BIGINT DEFAULT 1;
  SET autocommit = 0;

  WHILE i <= 1000000 DO
    -- 每个用户分配 1~5 个角色
    INSERT IGNORE INTO sys_user_role(user_id, role_id)
    VALUES
      (i, 1 + FLOOR(RAND() * 100)),
      (i, 1 + FLOOR(RAND() * 100)),
      (i, 1 + FLOOR(RAND() * 100));

    IF i MOD 5000 = 0 THEN
      COMMIT;
    END IF;
    SET i = i + 1;
  END WHILE;
  COMMIT;
END$$
DELIMITER ;

CALL insert_user_roles();
DROP PROCEDURE insert_user_roles;
```

### 3.4 生成 500 万登录日志

```sql
DELIMITER $$
CREATE PROCEDURE insert_login_logs()
BEGIN
  DECLARE i BIGINT DEFAULT 1;
  SET autocommit = 0;

  WHILE i <= 5000000 DO
    INSERT INTO sys_user_login_log(
      user_id, username, login_ip, login_time, logout_time,
      device_type, login_result, fail_reason
    ) VALUES (
      1 + FLOOR(RAND() * 1000000),
      CONCAT('user_', LPAD(1 + FLOOR(RAND() * 1000000), 7, '0')),
      CONCAT(
        1 + FLOOR(RAND() * 223), '.',
        FLOOR(RAND() * 256), '.',
        FLOOR(RAND() * 256), '.',
        FLOOR(RAND() * 256)
      ),
      DATE_SUB(NOW(), INTERVAL FLOOR(RAND() * 365 * 24 * 60) MINUTE),
      IF(RAND() > 0.05,
         DATE_SUB(NOW(), INTERVAL FLOOR(RAND() * 365 * 24 * 60 - 60) MINUTE),
         NULL),
      1 + FLOOR(RAND() * 3),
      IF(RAND() > 0.1, 1, 0),
      IF(RAND() > 0.9, '密码错误', NULL)
    );

    IF i MOD 5000 = 0 THEN
      COMMIT;
    END IF;
    SET i = i + 1;
  END WHILE;
  COMMIT;
END$$
DELIMITER ;

CALL insert_login_logs();
DROP PROCEDURE insert_login_logs;
```

### 3.5 验证数据量

```sql
SELECT
  TABLE_NAME                              AS '表名',
  TABLE_ROWS                              AS '估算行数',
  ROUND(DATA_LENGTH / 1024 / 1024, 1)    AS '数据(MB)',
  ROUND(INDEX_LENGTH / 1024 / 1024, 1)   AS '索引(MB)'
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'rbac_practice'
ORDER BY TABLE_ROWS DESC;
```

---

## 第四章：实战一：全表扫描 → 补索引

> **目标**：亲眼看到全表扫描有多慢，补上索引后性能提升多少倍。

### 4.1 场景：按用户名查询

这是最常见的场景——用户登录时，用 username 查询用户信息。

**第一步：先观察没有索引时的情况**

```sql
-- 查看当前 sys_user 的索引（应该只有主键）
SHOW INDEX FROM sys_user;

-- 执行查询，记录耗时
SELECT * FROM sys_user WHERE username = 'user_0500000';
```

```sql
-- 用 EXPLAIN 看执行计划
EXPLAIN SELECT * FROM sys_user WHERE username = 'user_0500000';
```

**预期结果（无索引时）：**

```
+----+-------------+----------+------+---------------+------+---------+------+---------+-------------+
| id | select_type | table    | type | possible_keys | key  | key_len | ref  | rows    | Extra       |
+----+-------------+----------+------+---------------+------+---------+------+---------+-------------+
|  1 | SIMPLE      | sys_user | ALL  | NULL          | NULL | NULL    | NULL | 1000000 | Using where |
+----+-------------+----------+------+---------------+------+---------+------+---------+-------------+
```

> ❗ `type = ALL`，`rows ≈ 100万`，全表扫描！耗时可能超过 1 秒。

**第二步：加索引**

```sql
-- 给 username 加唯一索引（业务上 username 应该唯一）
ALTER TABLE sys_user ADD UNIQUE KEY uk_username (username);
```

**第三步：再次 EXPLAIN 对比**

```sql
EXPLAIN SELECT * FROM sys_user WHERE username = 'user_0500000';
```

**预期结果（有索引后）：**

```
+----+-------------+----------+-------+---------------+-------------+---------+-------+------+-------+
| id | select_type | table    | type  | possible_keys | key         | key_len | ref   | rows | Extra |
+----+-------------+----------+-------+---------------+-------------+---------+-------+------+-------+
|  1 | SIMPLE      | sys_user | const | uk_username   | uk_username | 202     | const |    1 |       |
+----+-------------+----------+-------+---------------+-------------+---------+-------+------+-------+
```

> ✅ `type = const`，`rows = 1`，毫秒级响应！

### 4.2 场景：按手机号查询

```sql
-- ❌ 无索引，全表扫描
EXPLAIN SELECT id, username, real_name FROM sys_user WHERE mobile = '13812345678';

-- 加索引
ALTER TABLE sys_user ADD UNIQUE KEY uk_mobile (mobile);

-- ✅ 再次查看
EXPLAIN SELECT id, username, real_name FROM sys_user WHERE mobile = '13812345678';
```

### 4.3 场景：按部门查询用户

```sql
-- 查询某部门下的所有用户
EXPLAIN SELECT * FROM sys_user WHERE dept_id = 100;
-- 预期：type=ALL，rows=100万（无索引）

-- 加索引
ALTER TABLE sys_user ADD KEY idx_dept_id (dept_id);

-- 再查
EXPLAIN SELECT * FROM sys_user WHERE dept_id = 100;
-- 预期：type=ref（dept_id 非唯一，可能返回多行）
```

### 4.4 小结：建索引原则

| 场景 | 建议索引类型 |
|------|------------|
| 登录名、手机号等唯一字段 | `UNIQUE KEY` |
| 外键字段（dept_id）| 普通 `KEY` |
| 高频查询的过滤字段 | 普通 `KEY` 或联合索引 |
| 很少查询的字段 | 不建，减少写入开销 |

---

## 第五章：实战二：联合索引设计与最左前缀

> **目标**：掌握联合索引的列顺序设计，理解最左前缀在真实 SQL 中的表现。

### 5.1 业务场景：用户列表查询

后台用户管理页面，常见查询条件组合：

```sql
-- 组合1：按部门 + 状态筛选
SELECT * FROM sys_user WHERE dept_id = 100 AND status = 1;

-- 组合2：按状态筛选 + 按创建时间排序
SELECT * FROM sys_user WHERE status = 1 ORDER BY create_time DESC LIMIT 20;

-- 组合3：按部门 + 状态 + 创建时间范围
SELECT * FROM sys_user
WHERE dept_id = 100 AND status = 1
  AND create_time >= '2024-01-01' AND create_time < '2025-01-01';
```

### 5.2 第一轮：分析应该建什么联合索引

**分析查询模式：**

| 查询 | WHERE 字段 | ORDER BY |
|------|-----------|---------|
| 组合1 | dept_id（等值）、status（等值）| 无 |
| 组合2 | status（等值）| create_time |
| 组合3 | dept_id（等值）、status（等值）、create_time（范围）| 无 |

**联合索引设计推导：**

- 等值查询列放前，范围查询列放后
- status 区分度低（只有 0/1/2），dept_id 区分度高，dept_id 优先
- 考虑覆盖 ORDER BY 的 create_time，放最后

```sql
-- 推荐：idx_dept_status_createtime(dept_id, status, create_time)
CREATE INDEX idx_dept_status_ct ON sys_user (dept_id, status, create_time);
```

### 5.3 第二轮：验证各查询的索引命中情况

```sql
-- 组合1：dept_id + status，命中前两列 ✅
EXPLAIN SELECT * FROM sys_user WHERE dept_id = 100 AND status = 1;

-- 组合3：dept_id + status + create_time 范围，全命中 ✅
EXPLAIN SELECT * FROM sys_user
WHERE dept_id = 100 AND status = 1
  AND create_time >= '2024-01-01' AND create_time < '2025-01-01';

-- 单独 status 查询，是否命中？❌
EXPLAIN SELECT * FROM sys_user WHERE status = 1;
-- 跳过了 dept_id，联合索引最左前缀不满足

-- dept_id 单独查询，命中！✅
EXPLAIN SELECT * FROM sys_user WHERE dept_id = 100;
```

**通过 key_len 推断命中了几列：**

```sql
EXPLAIN SELECT * FROM sys_user WHERE dept_id = 100 AND status = 1;
-- 查看 key_len 值：
-- dept_id INT UNSIGNED = 4 bytes
-- status TINYINT NOT NULL = 1 byte
-- 如果 key_len = 5，说明命中了 dept_id + status 两列 ✅
```

### 5.4 陷阱：范围查询后的列无法利用索引

```sql
-- 索引：(dept_id, status, create_time)
-- dept_id 范围查询 → status、create_time 都失效
EXPLAIN SELECT * FROM sys_user
WHERE dept_id > 500 AND status = 1;
-- 实际只用到 dept_id 列，status 无法用索引过滤
```

**解法：** 将等值列放前，范围列放后，或拆分为两个索引。

---

## 第六章：实战三：覆盖索引消除回表

> **目标**：通过覆盖索引让查询完全在索引层完成，消除回表开销。

### 6.1 场景：用户列表分页（只显示部分字段）

常见的列表页，只需展示 id、username、real_name、dept_id、status，不需要所有字段：

```sql
-- 当前索引：idx_dept_status_ct(dept_id, status, create_time)
-- 查询
SELECT id, username, real_name, dept_id, status
FROM sys_user
WHERE dept_id = 100 AND status = 1
ORDER BY create_time DESC
LIMIT 20;
```

**查看执行计划：**

```sql
EXPLAIN SELECT id, username, real_name, dept_id, status
FROM sys_user
WHERE dept_id = 100 AND status = 1
ORDER BY create_time DESC
LIMIT 20;
-- Extra: Using index condition（有回表）
-- 因为 username、real_name 不在索引中，需要回聚簇索引取完整行
```

**优化：把查询列加入索引，实现覆盖索引**

```sql
-- 方案：加一个包含 username、real_name 的覆盖索引
-- 注意：列顺序 = WHERE 列 + ORDER BY 列 + SELECT 列
CREATE INDEX idx_dept_status_ct_cover
  ON sys_user (dept_id, status, create_time, username, real_name);
```

```sql
-- 再次 EXPLAIN
EXPLAIN SELECT id, username, real_name, dept_id, status
FROM sys_user
WHERE dept_id = 100 AND status = 1
ORDER BY create_time DESC
LIMIT 20;
-- Extra: Using index ← 覆盖索引！无需回表 ✅
```

### 6.2 覆盖索引 vs 回表性能对比

```sql
-- 对比测试（先关闭查询缓存）
SET SESSION query_cache_type = 0;

-- 方式1：强制走主键（全表扫描，模拟无索引）
SELECT SQL_NO_CACHE id, username, real_name, dept_id, status
FROM sys_user FORCE INDEX (PRIMARY)
WHERE dept_id = 100 AND status = 1
LIMIT 20;

-- 方式2：走覆盖索引
SELECT SQL_NO_CACHE id, username, real_name, dept_id, status
FROM sys_user USE INDEX (idx_dept_status_ct_cover)
WHERE dept_id = 100 AND status = 1
ORDER BY create_time DESC
LIMIT 20;
```

### 6.3 什么时候不适合覆盖索引

- SELECT 列太多（如 SELECT *）→ 索引太宽，写入开销大
- 字段更新频繁（每次更新都要维护宽索引）
- 索引已经满足过滤，回表次数本来就很少

---

## 第七章：实战四：深分页优化

> **目标**：理解为什么 `LIMIT 100万, 20` 极慢，掌握两种优化方案。

### 7.1 复现深分页慢查询

```sql
-- 第一页，非常快
SELECT id, username, real_name, status
FROM sys_user
WHERE status = 1
ORDER BY id
LIMIT 0, 20;

-- 第 5 万页（跳过 100 万行），极慢！
SELECT id, username, real_name, status
FROM sys_user
WHERE status = 1
ORDER BY id
LIMIT 1000000, 20;

-- EXPLAIN 分析
EXPLAIN SELECT id, username, real_name, status
FROM sys_user
WHERE status = 1
ORDER BY id
LIMIT 1000000, 20;
-- rows 仍然是 100 万，MySQL 要扫描并丢弃 100 万行，只返回最后 20 行！
```

### 7.2 优化方案一：子查询定位（任意跳页）

```sql
-- 核心思路：先用覆盖索引找到目标页的 id 范围，再用主键查完整数据
SELECT u.*
FROM sys_user u
INNER JOIN (
  SELECT id
  FROM sys_user
  WHERE status = 1
  ORDER BY id
  LIMIT 1000000, 20       -- 内层只查 id，走覆盖索引，速度快
) t ON u.id = t.id;
```

**EXPLAIN 对比：**

```sql
EXPLAIN SELECT u.*
FROM sys_user u
INNER JOIN (
  SELECT id FROM sys_user WHERE status = 1 ORDER BY id LIMIT 1000000, 20
) t ON u.id = t.id;
-- 内层子查询：type=index，Extra=Using index（覆盖索引）
-- 外层：type=eq_ref（主键等值 JOIN），极快
```

### 7.3 优化方案二：游标翻页（连续翻页，性能最优）

```sql
-- 记录上一页最后一条记录的 id，下次从此 id 之后开始查
-- 第一页
SELECT id, username, real_name, status
FROM sys_user
WHERE status = 1 AND id > 0
ORDER BY id
LIMIT 20;
-- 假设最后一条的 id = 982753

-- 第二页（游标继续）
SELECT id, username, real_name, status
FROM sys_user
WHERE status = 1 AND id > 982753
ORDER BY id
LIMIT 20;
-- 无论翻到第几页，都只扫描 20 行！
```

### 7.4 两种方案对比

| 方案 | 适用场景 | 性能 | 缺点 |
|------|---------|------|------|
| 子查询定位 | 后台管理，支持任意跳页 | 中等 | 实现稍复杂，仍有内层扫描开销 |
| 游标翻页 | 移动端/APP 无限滚动、下一页 | 最优（O(1)）| 不支持跳页，需记录游标值 |

---

## 第八章：实战五：JOIN 多表关联优化

> **目标**：掌握 JOIN 中驱动表选择、关联字段索引、字符集一致性的优化技巧。

### 8.1 场景：查询用户及其角色列表

```sql
-- 当前 sys_user_role 没有任何索引
EXPLAIN
SELECT u.id, u.username, u.real_name, r.role_name, r.role_code
FROM sys_user u
INNER JOIN sys_user_role ur ON ur.user_id = u.id
INNER JOIN sys_role r       ON r.id = ur.role_id
WHERE u.status = 1
  AND u.dept_id = 100
LIMIT 100;
```

**预期问题：**
- `sys_user_role` 无索引 → `type=ALL`，扫描 300 万行
- `Extra: Using join buffer` → 没用到索引，用了内存 buffer

**加索引优化：**

```sql
-- 关联表的关联字段必须有索引！
ALTER TABLE sys_user_role ADD KEY idx_user_id (user_id);
ALTER TABLE sys_user_role ADD KEY idx_role_id (role_id);

-- 再次 EXPLAIN
EXPLAIN
SELECT u.id, u.username, u.real_name, r.role_name, r.role_code
FROM sys_user u
INNER JOIN sys_user_role ur ON ur.user_id = u.id
INNER JOIN sys_role r       ON r.id = ur.role_id
WHERE u.status = 1 AND u.dept_id = 100
LIMIT 100;
-- ur: type=ref，key=idx_user_id ✅
-- r:  type=eq_ref，key=PRIMARY  ✅
```

### 8.2 验证驱动表顺序的影响

MySQL 优化器会选择最优驱动表，也可以手动验证：

```sql
-- 强制用 sys_user 作驱动表（小结果集驱动大表，正确）
EXPLAIN
SELECT STRAIGHT_JOIN u.id, u.username, ur.role_id
FROM sys_user u
STRAIGHT_JOIN sys_user_role ur ON ur.user_id = u.id
WHERE u.dept_id = 100;

-- 强制用 sys_user_role 作驱动表（大表驱动小表，错误示范）
EXPLAIN
SELECT STRAIGHT_JOIN u.id, u.username, ur.role_id
FROM sys_user_role ur
STRAIGHT_JOIN sys_user u ON u.id = ur.user_id
WHERE u.dept_id = 100;
```

> 💡 **结论**：小结果集（过滤后的 sys_user）驱动大结果集（sys_user_role），是正确的驱动方向。MySQL 优化器通常能自动判断，但复杂 SQL 可能需要手动干预。

### 8.3 查询用户的所有权限（三表 JOIN）

```sql
-- 通过 用户 → 角色 → 权限 的链路查出所有权限
SELECT DISTINCT p.perm_code, p.perm_name, p.perm_type
FROM sys_user u
INNER JOIN sys_user_role    ur ON ur.user_id     = u.id
INNER JOIN sys_role_permission rp ON rp.role_id   = ur.role_id
INNER JOIN sys_permission   p  ON p.id            = rp.permission_id
WHERE u.id = 12345;
```

```sql
-- EXPLAIN 分析
EXPLAIN SELECT DISTINCT p.perm_code, p.perm_name, p.perm_type
FROM sys_user u
INNER JOIN sys_user_role    ur ON ur.user_id     = u.id
INNER JOIN sys_role_permission rp ON rp.role_id   = ur.role_id
INNER JOIN sys_permission   p  ON p.id            = rp.permission_id
WHERE u.id = 12345;
```

---

## 第九章：实战六：ORDER BY / GROUP BY 优化

> **目标**：消除 `Using filesort` 和 `Using temporary`，让排序直接走索引。

### 9.1 场景：ORDER BY 优化

```sql
-- 索引：idx_dept_status_ct(dept_id, status, create_time)

-- ✅ 走索引排序（WHERE 等值列 + ORDER BY 恰好是索引前缀）
EXPLAIN SELECT id, username, create_time
FROM sys_user
WHERE dept_id = 100 AND status = 1
ORDER BY create_time DESC;
-- Extra: Using index condition（无 filesort）✅

-- ❌ filesort（ORDER BY 字段不在索引中）
EXPLAIN SELECT id, username, create_time
FROM sys_user
WHERE dept_id = 100
ORDER BY real_name;
-- Extra: Using filesort ❌

-- ❌ 排序方向混合（索引只支持同向排序）
EXPLAIN SELECT id, dept_id, status
FROM sys_user
WHERE dept_id = 100
ORDER BY status ASC, create_time DESC;
-- Extra: Using filesort ❌（一个 ASC 一个 DESC，无法走索引）
```

### 9.2 场景：GROUP BY 统计优化

```sql
-- 统计各部门的用户数
EXPLAIN
SELECT dept_id, COUNT(*) AS user_count
FROM sys_user
WHERE status = 1
GROUP BY dept_id;
-- 如果 dept_id 有索引，可以利用索引分组，避免 Using temporary

-- 统计各状态的用户数（status 只有 3 个值）
EXPLAIN
SELECT status, COUNT(*) AS cnt
FROM sys_user
GROUP BY status;
```

**为 GROUP BY 加索引：**

```sql
-- 统计各部门各状态的用户数
EXPLAIN
SELECT dept_id, status, COUNT(*) AS cnt
FROM sys_user
GROUP BY dept_id, status;
-- 如果有 idx_dept_status_ct 索引，GROUP BY (dept_id, status) 可以走索引
-- Extra: Using index ✅
```

### 9.3 登录日志统计（日志大表 GROUP BY）

```sql
-- 统计每个用户的登录次数（500 万条日志表，无索引）
EXPLAIN
SELECT user_id, COUNT(*) AS login_cnt
FROM sys_user_login_log
WHERE login_time >= '2024-01-01'
GROUP BY user_id
ORDER BY login_cnt DESC
LIMIT 10;
-- 预期：type=ALL，Using temporary，Using filesort，极慢！
```

**加索引优化：**

```sql
-- 登录日志表：常用查询维度是 user_id 和 login_time
ALTER TABLE sys_user_login_log
  ADD KEY idx_user_login_time (user_id, login_time);

-- 也可以为时间范围查询单独加索引
ALTER TABLE sys_user_login_log
  ADD KEY idx_login_time (login_time);

-- 再次 EXPLAIN
EXPLAIN
SELECT user_id, COUNT(*) AS login_cnt
FROM sys_user_login_log
WHERE login_time >= '2024-01-01'
GROUP BY user_id
ORDER BY login_cnt DESC
LIMIT 10;
```

---

## 第十章：实战七：索引失效场景排查

> **目标**：在真实数据上亲手复现各种索引失效场景，加深记忆。

### 10.1 场景一：对索引列做函数

```sql
-- 登录日志表已有 idx_login_time(login_time)

-- ❌ 对 login_time 用 DATE 函数，索引失效
EXPLAIN SELECT * FROM sys_user_login_log
WHERE DATE(login_time) = '2024-06-01';
-- type=ALL，全表扫描 500 万行 ❌

-- ✅ 改写为范围查询
EXPLAIN SELECT * FROM sys_user_login_log
WHERE login_time >= '2024-06-01 00:00:00'
  AND login_time <  '2024-06-02 00:00:00';
-- type=range，走 idx_login_time ✅
```

### 10.2 场景二：隐式类型转换

```sql
-- sys_user 表 mobile 是 VARCHAR(20)

-- ❌ 传入数字，触发隐式转换，索引失效
EXPLAIN SELECT * FROM sys_user WHERE mobile = 13800000000;
-- type=ALL ❌

-- ✅ 传入字符串，类型一致，索引生效
EXPLAIN SELECT * FROM sys_user WHERE mobile = '13800000000';
-- type=const 或 ref ✅
```

### 10.3 场景三：LIKE 前缀通配符

```sql
-- ❌ 前缀通配，无法从 B+ 树定位起点
EXPLAIN SELECT * FROM sys_user WHERE username LIKE '%_050%';
-- type=ALL ❌

-- ✅ 后缀通配，可以利用索引
EXPLAIN SELECT * FROM sys_user WHERE username LIKE 'user_050%';
-- type=range ✅
```

### 10.4 场景四：OR 中存在非索引列

```sql
-- username 有唯一索引，real_name 没有索引
-- ❌ OR 中 real_name 无索引，拖累整个查询走全表
EXPLAIN SELECT * FROM sys_user
WHERE username = 'user_0500000' OR real_name = '王伟';
-- type=ALL ❌

-- ✅ 方案一：给 real_name 加索引
ALTER TABLE sys_user ADD KEY idx_real_name (real_name);

-- ✅ 方案二：改写为 UNION（各自走索引）
EXPLAIN
SELECT * FROM sys_user WHERE username = 'user_0500000'
UNION
SELECT * FROM sys_user WHERE real_name = '王伟';
```

### 10.5 场景五：联合索引字段类型不一致

```sql
-- 假设 sys_user.dept_id 是 INT UNSIGNED
-- sys_dept.id 也是 INT UNSIGNED，类型一致，索引生效 ✅

-- 模拟反例：如果两表字符集不同，JOIN 时索引失效
-- 查看表字符集
SHOW CREATE TABLE sys_user\G
SHOW CREATE TABLE sys_dept\G
-- 确保都是 utf8mb4，否则 JOIN 时字符集隐式转换会导致索引失效
```

### 10.6 综合排查：一条难找的失效 SQL

```sql
-- 看这条 SQL，能发现几个问题？
SELECT *
FROM sys_user
WHERE IFNULL(dept_id, 0) = 100
  AND status != 0
  AND CONCAT(real_name, '') LIKE '%张%'
ORDER BY RAND();
```

**逐一分析：**

| 问题 | 原因 | 修复思路 |
|------|------|---------|
| `IFNULL(dept_id, 0)` | 对索引列用了函数，dept_id 索引失效 | 改为 `dept_id = 100`（dept_id 不为 NULL）|
| `status != 0` | 负向查询，索引可能失效 | 改为 `status IN (1, 2)` |
| `CONCAT(real_name, '')` | 对索引列做函数，索引失效 | 先别想索引，这个需要全文检索 |
| `ORDER BY RAND()` | 随机排序，无法利用任何索引，filesort | 业务层随机化，不要在 SQL 里 RAND() |

---

## 第十一章：综合练习题

> 以下为综合练习，建议先自己写出解答，再看分析。

### 练习 1：设计用户搜索接口的索引

**需求**：后台用户列表支持以下筛选条件（可多选）：
- 部门（dept_id）
- 状态（status）
- 注册时间范围（create_time）
- 用户名模糊搜索（username LIKE）

请设计索引方案，并用 EXPLAIN 验证。

<details>
<summary>参考答案</summary>

```sql
-- 等值字段在前，范围字段在后
-- 模糊查询用后缀 LIKE，可以利用索引
CREATE INDEX idx_dept_status_ct ON sys_user (dept_id, status, create_time);

-- 如果 username 有前缀 LIKE 需求，单独建索引
-- uk_username 已存在，LIKE 'user_%' 可以走这个索引

-- 验证
EXPLAIN SELECT id, username, real_name, dept_id, status, create_time
FROM sys_user
WHERE dept_id = 100
  AND status = 1
  AND create_time >= '2024-01-01'
  AND username LIKE 'user_05%'
ORDER BY create_time DESC
LIMIT 20;
```

</details>

---

### 练习 2：优化登录日志统计慢查询

**需求**：统计最近 30 天内，登录失败超过 5 次的用户 ID 列表。

```sql
-- 当前 SQL（很慢）
SELECT user_id, COUNT(*) AS fail_cnt
FROM sys_user_login_log
WHERE login_time >= DATE_SUB(NOW(), INTERVAL 30 DAY)
  AND login_result = 0
GROUP BY user_id
HAVING fail_cnt >= 5
ORDER BY fail_cnt DESC;
```

请：
1. EXPLAIN 分析当前性能瓶颈
2. 设计索引方案
3. 用 EXPLAIN 验证优化后的效果

<details>
<summary>参考答案</summary>

```sql
-- 分析：WHERE 有 login_time（范围）和 login_result（等值）+ GROUP BY user_id
-- login_result 区分度低（只有 0/1），login_time 区分度高
-- 索引设计：把等值放前，范围放后 → (login_result, login_time, user_id)
-- user_id 放最后，实现覆盖索引，GROUP BY 不需要回表

ALTER TABLE sys_user_login_log
  ADD KEY idx_result_time_user (login_result, login_time, user_id);

EXPLAIN
SELECT user_id, COUNT(*) AS fail_cnt
FROM sys_user_login_log
WHERE login_time >= DATE_SUB(NOW(), INTERVAL 30 DAY)
  AND login_result = 0
GROUP BY user_id
HAVING fail_cnt >= 5
ORDER BY fail_cnt DESC;
-- 期望：type=range，key=idx_result_time_user，Extra=Using index
```

</details>

---

### 练习 3：优化权限查询接口

**需求**：用户每次请求都需要查询其所有权限编码（用于鉴权），这个查询要极快。

```sql
-- 当前 SQL
SELECT DISTINCT p.perm_code
FROM sys_user_role ur
INNER JOIN sys_role_permission rp ON rp.role_id = ur.role_id
INNER JOIN sys_permission p       ON p.id = rp.permission_id
WHERE ur.user_id = ?;
```

请分析性能瓶颈并给出优化方案。

<details>
<summary>参考答案</summary>

```sql
-- 1. 确保关联字段有索引
SHOW INDEX FROM sys_user_role;        -- 需要 idx_user_id
SHOW INDEX FROM sys_role_permission;  -- 已有 uk_role_perm, idx_permission_id
SHOW INDEX FROM sys_permission;       -- 已有主键

-- 2. sys_user_role 加 user_id 索引（前面已加）
-- ALTER TABLE sys_user_role ADD KEY idx_user_id (user_id);

-- 3. 考虑覆盖索引：sys_user_role 只用到 user_id 和 role_id，用联合索引覆盖
ALTER TABLE sys_user_role
  ADD KEY idx_user_role_cover (user_id, role_id);

-- 4. 鉴权场景建议引入 Redis 缓存，将用户权限集合缓存，避免每次查库
-- 只在权限变更时刷新缓存

-- EXPLAIN 验证
EXPLAIN SELECT DISTINCT p.perm_code
FROM sys_user_role ur
INNER JOIN sys_role_permission rp ON rp.role_id = ur.role_id
INNER JOIN sys_permission p       ON p.id = rp.permission_id
WHERE ur.user_id = 12345;
```

</details>

---

### 练习 4：找出慢查询并优化

以下 SQL 是真实运行的，每一条都有性能问题，请找出来并修复：

```sql
-- SQL 1
SELECT * FROM sys_user WHERE DATE(create_time) = '2024-06-01';

-- SQL 2
SELECT * FROM sys_user
WHERE status = 1 ORDER BY RAND() LIMIT 10;

-- SQL 3
SELECT * FROM sys_user_login_log
WHERE user_id IN (
  SELECT id FROM sys_user WHERE dept_id = 100 AND status = 1
)
AND login_time > '2024-01-01';

-- SQL 4
SELECT dept_id, COUNT(*) FROM sys_user
WHERE status = 1
GROUP BY dept_id
HAVING COUNT(*) > 50
ORDER BY COUNT(*) DESC;
```

<details>
<summary>参考答案</summary>

```sql
-- SQL 1 修复：函数导致索引失效，改为范围查询
ALTER TABLE sys_user ADD KEY idx_create_time (create_time);
SELECT * FROM sys_user
WHERE create_time >= '2024-06-01 00:00:00'
  AND create_time <  '2024-06-02 00:00:00';

-- SQL 2 修复：ORDER BY RAND() 极慢，业务层随机化
-- 方案：先取总数，随机offset，再 LIMIT 1（或多次查询）
SELECT * FROM sys_user WHERE status = 1 AND id >= (
  SELECT FLOOR(RAND() * (SELECT MAX(id) FROM sys_user WHERE status=1))
) LIMIT 10;

-- SQL 3 修复：子查询改为 JOIN，让优化器选择最优执行顺序
SELECT l.*
FROM sys_user u
INNER JOIN sys_user_login_log l ON l.user_id = u.id
WHERE u.dept_id = 100
  AND u.status = 1
  AND l.login_time > '2024-01-01';
-- 确保 l.user_id 和 l.login_time 有索引

-- SQL 4 修复：已有 idx_dept_status_ct(dept_id, status, create_time)
-- GROUP BY dept_id WHERE status=1，索引中 status 在 dept_id 后，需要调整
-- 重建索引：status 作为等值列，放 dept_id 前
CREATE INDEX idx_status_dept ON sys_user (status, dept_id);
SELECT dept_id, COUNT(*) FROM sys_user
WHERE status = 1
GROUP BY dept_id
HAVING COUNT(*) > 50
ORDER BY COUNT(*) DESC;
```

</details>

---

## 附录：快速参考

### 常用 EXPLAIN 分析 SQL

```sql
-- 查看某张表的所有索引
SHOW INDEX FROM sys_user;

-- 查看某 SQL 的执行计划
EXPLAIN SELECT ...;

-- MySQL 8.0 查看实际执行情况（包含真实耗时和行数）
EXPLAIN ANALYZE SELECT ...;

-- 查看优化器为什么没选某个索引
SET optimizer_trace='enabled=on';
SELECT ...;
SELECT * FROM information_schema.OPTIMIZER_TRACE\G
SET optimizer_trace='enabled=off';
```

### 各表推荐索引汇总

```sql
-- sys_user（用户表，100万）
ALTER TABLE sys_user ADD UNIQUE KEY uk_username    (username);
ALTER TABLE sys_user ADD UNIQUE KEY uk_mobile      (mobile);
ALTER TABLE sys_user ADD KEY        idx_dept_id    (dept_id);
ALTER TABLE sys_user ADD KEY        idx_dept_status_ct (dept_id, status, create_time);

-- sys_user_role（用户角色关联，300万）
ALTER TABLE sys_user_role ADD KEY idx_user_id  (user_id);
ALTER TABLE sys_user_role ADD KEY idx_role_id  (role_id);

-- sys_user_login_log（登录日志，500万）
ALTER TABLE sys_user_login_log ADD KEY idx_user_id    (user_id);
ALTER TABLE sys_user_login_log ADD KEY idx_login_time (login_time);
ALTER TABLE sys_user_login_log ADD KEY idx_result_time_user (login_result, login_time, user_id);
```

### 性能问题快速诊断

```sql
-- 找出正在执行的慢 SQL
SELECT * FROM information_schema.PROCESSLIST
WHERE COMMAND != 'Sleep' AND TIME > 5
ORDER BY TIME DESC;

-- 找出历史执行最慢的 SQL 类型
SELECT DIGEST_TEXT,
       COUNT_STAR             AS exec_count,
       ROUND(AVG_TIMER_WAIT/1e9, 2) AS avg_ms,
       ROUND(SUM_TIMER_WAIT/1e9, 2) AS total_ms
FROM performance_schema.events_statements_summary_by_digest
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 10;
```

---

> **学习路径建议**：
> 1. 先按顺序完成 4~10 章的每个实战
> 2. 然后独立完成第 11 章全部练习题（不看答案）
> 3. 修改测试数据量（试试 10 万、500 万），观察性能变化规律
> 4. 结合《MySQL专项学习手册（完整版）》对照理论，达到知行合一
