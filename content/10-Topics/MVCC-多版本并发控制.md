---
tags:
  - topic/数据库事务
  - status/reviewed
created: 2026-05-23
---

# MVCC（多版本并发控制）

## 什么是MVCC？

MVCC（Multi-Version Concurrency Control，多版本并发控制）是一种用于解决数据库并发访问问题的技术。它通过为每个事务提供数据在某个时间点的快照（Snapshot），使得读操作不会被写操作阻塞，实现了**读写不冲突**的并发控制方式。

## 核心思想

MVCC的核心思想是：**同一份数据可以被同时存在多个版本，每个事务看到的都是符合自己"观察时间点"的一致性视图**。

### 版本链（Version Chain）

在InnoDB中，每行数据实际上包含两个隐藏字段：

| 隐藏字段 | 说明 |
|---------|------|
| `trx_id` | 最近修改该行数据的事务ID |
| `roll_pointer` | 指向undo log的指针，形成版本链 |

当数据被修改时：

```
初始数据（trx_id=100）→ 修改后（trx_id=101）→ 再次修改（trx_id=102）
    ↑                      ↑                      ↑
  undo log              undo log              当前数据
```

## Read View机制

### 什么是Read View？

Read View（读视图）是事务在开启瞬间创建的"快照"，包含以下关键信息：

| 字段 | 说明 |
|------|------|
| `m_ids` | 当前活跃事务ID列表（未提交的事务） |
| `min_trx_id` | 活跃事务中的最小ID |
| `max_trx_id` | 创建Read View时最大事务ID + 1 |
| `creator_trx_id` | 当前事务自身的事务ID |

### 可见性判断规则

当事务读取某行数据时，按照以下规则判断该数据是否可见：

```
1. 如果数据的 trx_id == creator_trx_id
   → 可见（自己修改的数据当然可见）

2. 如果数据的 trx_id < min_trx_id
   → 可见（说明该事务已提交）

3. 如果数据的 trx_id >= max_trx_id
   → 不可见（该事务在Read View创建之后才开启）

4. 如果数据的 trx_id 在 m_ids 列表中
   → 不可见（该事务还未提交）
```

## 瑞幸场景示例

### 场景：查询用户咖啡券余额

**业务背景：** 用户在查看咖啡券余额的同时，会员系统可能正在更新用户的等级和积分。

### 无MVCC（读已提交 + 锁）

```
T1: 事务A开启，查询用户余额
    → SELECT * FROM coupon WHERE user_id = 1;
    → 返回余额 = 5张（加读锁）

T2: 事务B更新用户余额
    → UPDATE coupon SET balance = 4 WHERE user_id = 1;
    → 被阻塞！等待事务A释放读锁

T3: 事务A再次查询余额
    → 返回余额 = 5张（与上次一致）

T4: 事务A提交，释放读锁

T5: 事务B获得写锁，更新成功

T6: 事务B查询余额
    → 返回余额 = 4张
```

**问题：** 事务B被阻塞，并发性能差。事务A多次读取结果一致，但代价是阻塞其他写操作。

### 有MVCC（读已提交）

```
T1: 事务A开启（trx_id=100），生成Read View
    → m_ids = [100]（自身）
    → min_trx_id = 100
    → max_trx_id = 101

T2: 事务A查询用户余额
    → SELECT balance FROM coupon WHERE user_id = 1;
    → 当前数据 trx_id=99 < min_trx_id(100)
    → 可见，返回余额 = 5张

T3: 事务B开启（trx_id=101），修改数据
    → UPDATE coupon SET balance = 4 WHERE user_id = 1;
    → 生成新版本数据，trx_id=101
    → 旧版本数据写入undo log

T4: 事务A再次查询余额
    → 当前数据 trx_id=101 在 m_ids 中
    → 不可见！沿着roll_pointer找到旧版本
    → 旧版本 trx_id=99 < min_trx_id(100)
    → 可见，返回余额 = 5张

T5: 事务B提交（trx_id=101已不在活跃列表）

T6: 事务A第三次查询余额
    → 当前数据 trx_id=101 < min_trx_id(102)（重新生成Read View）
    → 可见，返回余额 = 4张
```

**优势：** 事务A的读操作完全不被阻塞，通过undo log看到数据的历史版本。

## MVCC与隔离级别

### MySQL InnoDB的实现

| 隔离级别 | MVCC实现方式 |
|---------|-------------|
| READ_UNCOMMITTED | 不使用MVCC，总是读取最新数据 |
| READ_COMMITTED | 每次SELECT生成新的Read View |
| REPEATABLE_READ | 事务开始时生成Read View，整个事务复用 |
| SERIALIZABLE | 使用锁，MVCC退化为两阶段锁 |

### 可重复读与读已提交的区别

```
可重复读（Repeatable Read）：
T1: 事务A开启，Read View固定
T2: 事务A查询 → 余额=5
T3: 事务B修改余额=4，提交
T4: 事务A查询 → 余额=5（Read View未变）
T5: 事务A查询 → 余额=5
→ 事务A整个生命周期内，读到的数据一致

读已提交（Read Committed）：
T1: 事务A开启
T2: 事务A查询 → 余额=5
T3: 事务B修改余额=4，提交
T4: 事务A查询 → 余额=4（重新生成Read View）
→ 每次查询可能看到不同结果
```

## MVCC的局限性

### 不能完全解决幻读

幻读是指同一事务内，两次查询返回的记录数不同。MVCC通过快照读可以避免大部分幻读问题，但对于**当前读**（SELECT ... FOR UPDATE）仍可能出现幻读。

### 当前读 vs 快照读

| 类型 | 说明 | 是否受MVCC保护 |
|------|------|--------------|
| 快照读 | 普通SELECT语句 | ✅ 是 |
| 当前读 | SELECT ... FOR UPDATE | ❌ 否 |
| 当前读 | INSERT/UPDATE/DELETE | ❌ 否 |

### 瑞幸场景：并发领取优惠券

```
T1: 事务A查询优惠券列表（快照读）
    → 返回10张可用优惠券

T2: 事务B也查询优惠券列表（快照读）
    → 也返回10张（快照相同）

T3: 事务A领取优惠券（当前读 + INSERT）
    → INSERT成功

T4: 事务B领取优惠券（当前读 + INSERT）
    → INSERT成功（两张券都领取了）
    → 如果用快照读，可能出现重复领取的问题
```

## MVCC与锁机制的关系

### 互补关系

| 特性 | MVCC | 锁机制 |
|------|------|--------|
| 作用对象 | 读操作 | 写操作 |
| 阻塞情况 | 读不阻塞读 | 写阻塞读/写 |
| 解决场景 | 并发读 | 并发写 |
| 性能影响 | 低 | 高 |

### 组合使用

现代数据库通常将MVCC和锁机制组合使用：

- **读操作**：使用MVCC提供快照读
- **写操作**：使用锁机制（行锁/表锁）
- **需要"最新数据"时**：使用当前读（显式加锁）

### 瑞幸库存扣减示例

```
场景：两个用户同时购买最后1杯咖啡

正确做法：使用当前读 + 锁
T1: SELECT * FROM stock WHERE coffee_id = 1 FOR UPDATE;
    → 获得行锁，返回库存=1

T2: SELECT * FROM stock WHERE coffee_id = 1 FOR UPDATE;
    → 被阻塞，等待锁

T1: UPDATE stock SET count = 0 WHERE coffee_id = 1;
    → 扣减成功

T1: COMMIT; → 释放锁

T2: SELECT * FROM stock WHERE coffee_id = 1 FOR UPDATE;
    → 获取锁，返回库存=0
    → 提示"库存不足"

错误做法：使用快照读
T1: SELECT count FROM stock WHERE coffee_id = 1;
    → 返回库存=1

T2: SELECT count FROM stock WHERE coffee_id = 1;
    → 也返回库存=1

T1: UPDATE stock SET count = 0 WHERE coffee_id = 1;
    → 扣减成功

T2: UPDATE stock SET count = -1 WHERE coffee_id = 1;
    → 扣减成功
    → 超卖！
```

## 总结

### MVCC的核心价值

1. **读写不冲突**：读操作不会被写操作阻塞
2. **提高并发性能**：多个读操作可以并行执行
3. **保证隔离性**：每个事务看到一致的快照数据

### MVCC的适用场景

- **读多写少**：适合高并发读取场景
- **需要快照一致性**：报表、数据统计
- **长事务**：避免长时间锁定数据

### MVCC的注意事项

- 不能完全替代锁机制
- 写操作仍需要加锁保护
- 需要合理管理undo log空间
- 当前读仍可能导致并发问题

***

## 相关链接

- [[隔离级别]] - 四种隔离级别详解
- [[锁机制实现详解]] - 数据库锁机制原理
- [[乐观锁]] - MVCC相关的乐观并发控制
- [[悲观锁]] - 传统悲观并发控制方式
