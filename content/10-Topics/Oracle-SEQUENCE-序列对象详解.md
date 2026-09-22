---
title: Oracle SEQUENCE 序列对象详解
aliases:
  - Oracle Sequence
  - Oracle 序列
  - CREATE SEQUENCE
tags:
  - topic/Oracle
  - topic/数据库
  - language/sql
status: to-review
created: 2026-07-16
---

# Oracle SEQUENCE 序列对象详解

**SEQUENCE（序列）** 是 Oracle 中一个独立于表的**数据库对象**，用于自动生成**唯一且递增（或递减）的数值序列**。它是 Oracle 实现自增主键的标准方案（Oracle 没有 MySQL 的 `AUTO_INCREMENT` 属性）。

---

## 基本语法

```sql
CREATE SEQUENCE sequence_name
  [START WITH  n]        -- 起始值，默认 1
  [INCREMENT BY n]       -- 步长，正数递增 / 负数递减，默认 1
  [MINVALUE n]           -- 最小值
  [MAXVALUE n]           -- 最大值
  [CACHE n | NOCACHE]    -- 是否缓存序列值到内存
  [CYCLE | NOCYCLE]      -- 达到极值后是否循环
  [ORDER | NOORDER];     -- 是否保证按请求顺序分配（RAC 环境）
```

### 你的语句逐句解析

```sql
CREATE SEQUENCE SEQ_ANALYSIS_H
START WITH 1              -- 从 1 开始
INCREMENT BY 1            -- 每次 +1
NOCACHE                   -- 不缓存，每次写磁盘（保证值不丢失）
NOCYCLE;                  -- 不循环，达到最大值后报错（保证不重复）
```

| 参数 | 取值 | 含义 | 默认值 |
|------|------|------|--------|
| `START WITH` | `1` | 序列起始值 | `1` |
| `INCREMENT BY` | `1` | 步长（正 = 递增，负 = 递减） | `1` |
| `NOCACHE` | — | 不预缓存序列值到内存 | `CACHE 20` |
| `NOCYCLE` | — | 达到最大值后报错而非循环 | `NOCYCLE` |

> 设计意图：该序列很可能用于 `ANALYSIS_H` 表的主键生成，**注重连续性和唯一性**（`NOCACHE` 避免数据库重启后序列号跳跃，`NOCYCLE` 防止主键冲突）。

---

## 核心参数详解

### 1. `CACHE` vs `NOCACHE`

```sql
CACHE 20    -- 默认，内存中预生成 20 个序列号
NOCACHE     -- 不缓存，每次都写磁盘
```

| 维度 | CACHE | NOCACHE |
|------|-------|---------|
| **性能** | 🚀 高（内存操作，减少磁盘 I/O） | 🐢 低（每次写 `SEQ$` 系统表） |
| **值丢失风险** | ⚠️ 数据库异常关闭时，缓存中未用完的序列号会丢失 | ✅ 不会丢失 |
| **应用场景** | 高并发 OLTP 系统 | 序列号连续性要求严格的场景（如财务流水号） |

> [!tip] 经典折中
> 大多数生产系统使用 `CACHE 20` 或 `CACHE 100` 换取性能。`NOCACHE` 仅在序列号连续性为硬性要求时使用。

### 2. `CYCLE` vs `NOCYCLE`

```sql
CYCLE     -- 达到 MAXVALUE 后从 START WITH 或 MINVALUE 重新开始
NOCYCLE   -- 达到 MAXVALUE 后报错 ORA-08004
```

- **主键场景**：必须用 `NOCYCLE`，否则循环值会与已有主键冲突
- **流水号场景**：如果位数有限且号码用完可复用，可考虑 `CYCLE`

### 3. 最大值与最小值

```sql
CREATE SEQUENCE seq_test
  MINVALUE 1
  MAXVALUE 99999
  CYCLE;
```

- `NUMBER` 类型的默认 `MAXVALUE` 为 `9999999999999999999999999999999999999999`（38 位 9），几乎不会达到
- 递减序列（`INCREMENT BY -1`）需要指定 `MAXVALUE` 和 `MINVALUE`

### 4. `ORDER` vs `NOORDER`（RAC 环境）

```sql
ORDER      -- 保证多节点间序列值按请求顺序分配（全局有序）
NOORDER    -- 默认值，RAC 各节点独立缓存，可能有乱序
```

- 单实例环境没有区别
- RAC（Real Application Cluster）多节点环境中，`ORDER` 会引入跨节点同步开销

---

## 如何使用 SEQUENCE

### 获取下一个值

```sql
SELECT SEQ_ANALYSIS_H.NEXTVAL FROM DUAL;
-- 返回 1, 2, 3, ...
```

### 获取当前值

```sql
SELECT SEQ_ANALYSIS_H.CURRVAL FROM DUAL;
```

> [!warning] `CURRVAL` 必须先调用 `NEXTVAL` 一次才能读取，否则报 `ORA-08002`

### 在 INSERT 中使用（主键生成）

```sql
INSERT INTO analysis_h (id, record_data)
VALUES (SEQ_ANALYSIS_H.NEXTVAL, 'some data');
```

### 作为默认值（Oracle 12c+）

```sql
CREATE TABLE analysis_h (
  id NUMBER DEFAULT SEQ_ANALYSIS_H.NEXTVAL PRIMARY KEY,
  record_data VARCHAR2(4000)
);
```

---

## 序列的管理操作

### 修改序列参数

```sql
ALTER SEQUENCE SEQ_ANALYSIS_H
  INCREMENT BY 100
  MAXVALUE 999999999;
```

> [!note] 修改限制
> - `START WITH` **不可修改**（只能 `DROP` 重建）
> - 修改 `INCREMENT BY` 是跳号的常用技巧：`INCREMENT BY 100` → `NEXTVAL` → `INCREMENT BY 1` 可快速推进序列值

### 查询序列状态（数据字典）

```sql
-- 查看当前会话的序列值
SELECT SEQ_ANALYSIS_H.CURRVAL FROM DUAL;

-- 查看序列定义
SELECT * FROM USER_SEQUENCES WHERE SEQUENCE_NAME = 'SEQ_ANALYSIS_H';

-- 或
SELECT * FROM DBA_SEQUENCES WHERE SEQUENCE_NAME = 'SEQ_ANALYSIS_H';
```

`USER_SEQUENCES` 输出字段：

| 字段 | 说明 |
|------|------|
| `SEQUENCE_NAME` | 序列名称 |
| `MIN_VALUE` | 最小值 |
| `MAX_VALUE` | 最大值 |
| `INCREMENT_BY` | 步长 |
| `CYCLE_FLAG` | 是否循环（Y/N） |
| `ORDER_FLAG` | 是否有序（Y/N） |
| `CACHE_SIZE` | 缓存大小 |
| `LAST_NUMBER` | 最后一次写入磁盘的值（含缓存中已分配但未使用的） |

### 删除序列

```sql
DROP SEQUENCE SEQ_ANALYSIS_H;
```

---

## 常见面试题

> **Q: Oracle 中如何实现自增主键？**

A: 使用 SEQUENCE，配合 INSERT 语句的 `SEQ.NEXTVAL` 或 Oracle 12c+ 的 `GENERATED AS IDENTITY`：
```sql
-- 方式一：传统方式
CREATE SEQUENCE SEQ_EMP_ID START WITH 1;
INSERT INTO emp (id, name) VALUES (SEQ_EMP_ID.NEXTVAL, 'Alice');

-- 方式二：Oracle 12c+ 标识列（内部仍依赖序列）
CREATE TABLE emp (
  id NUMBER GENERATED AS IDENTITY PRIMARY KEY,
  name VARCHAR2(100)
);
```

> **Q: `NEXTVAL` 和 `CURRVAL` 的使用限制？**

A: 在以下场景中两者都**不可使用**：
- 在 `WHERE` 子句中
- 在 `CHECK` 约束中
- 在 `VIEW` 查询中（但可以在视图的 INSERT 中使用）
- 在子查询中

> **Q: 数据库重启后序列会跳号吗？**

A: 取决于 `CACHE`/`NOCACHE`：
- `CACHE n`：数据库异常关闭后，缓存中已分配但未使用的序列号**丢失**，下一轮从 `LAST_NUMBER`（已写盘的最后一个值）开始，产生**跳跃**
- `NOCACHE`：每次值都写磁盘，重启后不跳跃（但性能较差）

> **Q: Oracle 的 SEQUENCE 和 MySQL 的 `AUTO_INCREMENT` 有什么区别？**

| 维度 | Oracle SEQUENCE | MySQL AUTO_INCREMENT |
|------|-----------------|---------------------|
| **本质** | 独立数据库对象 | 表字段属性 |
| **跨表共享** | ✅ 多个表可共享一个序列 | ❌ 每个表独立 |
| **步长控制** | ✅ `INCREMENT BY` | ✅ `auto_increment_increment` |
| **缓存控制** | ✅ CACHE/NOCACHE | 由存储引擎决定（InnoDB 自缓存） |
| **事务回滚** | ❌ `NEXTVAL` 不回滚（值被永久消耗） | ❌ 不回滚 |
| **并发性能** | 高（可缓存） | 高（内存计数器） |

---

## 相关笔记

- [[唯一约束与唯一索引的区别-Oracle-Dm]] — Oracle/达梦数据库中约束与索引的差异
- [[主键索引与唯一索引的区别]] — 主键 vs 唯一索引对比
- [[隔离级别]] — Oracle 默认的 READ COMMITTED 隔离级别及相关特性
