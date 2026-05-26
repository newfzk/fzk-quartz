---
title: CAS（Compare And Swap）机制
date: 2026-05-23
tags:
  - topic/并发控制
  - topic/CAS
  - language/java
status: to-review
updated: 2026-05-26 15:30:00
aliases:
  - Compare-And-Swap
  - CAS机制
  - 乐观锁/CAS
related:
  - "[[乐观锁]]"
  - "[[MVCC-多版本并发控制]]"
  - "[[悲观锁]]"
  - "[[锁机制实现详解]]"
---

# CAS（Compare And Swap）机制

## 什么是CAS？

CAS（Compare And Swap，比较并交换）是一种**无锁并发控制算法**，通过硬件级别的原子操作实现乐观锁。

> 需要**注意，CAS仅是服务于java进程的防并发，与数据库事务无关**

### 核心原理

CAS操作包含三个参数：

- **内存位置**（V）：要更新的变量地址
- **期望值**（E）：读取的旧值
- **新值**（N）：要写入的新值

```
CAS(V, E, N) 执行过程：
┌─────────────────────────────────┐
│  1. 读取 V 的当前值             │
│     ↓                           │
│  2. 比较 当前值 == 期望值E？    │
│     ↓                           │
│  3. 如果相等 → 将 V 设为新值N   │
│     如果不等 → 操作失败，返回false│
└─────────────────────────────────┘
```

### CPU级别的原子保证

CAS是CPU提供的**硬件级原子指令**，确保比较和交换操作不可中断，避免了软件锁的开销。

## CAS与乐观锁的关系

### 核心关系图

```
乐观锁（理论/思想）
    ├── 版本号机制（数据库实现）
    ├── 时间戳机制（数据库实现）
    └── CAS机制（CPU/编程语言实现）← Java的AtomicInteger等原子类
```

### 对应关系

| 乐观锁（数据库）          | CAS（Java）                   |
| ----------------- | --------------------------- |
| 版本号 version       | 当前值 current                 |
| WHERE version = ? | compareAndSet(current, new) |
| 更新失败行数=0          | compareAndSet返回false        |
| 需要重试              | 重新循环                        |

## Java中的CAS实现

### AtomicInteger 示例

```java
import java.util.concurrent.atomic.AtomicInteger;

public class Counter {
    // 使用AtomicInteger替代普通int，保证原子性
    private AtomicInteger count = new AtomicInteger(0);
    
    public int increment() {
        int current;
        int next;
        do {
            current = count.get();      // 读取当前值
            next = current + 1;         // 计算新值
        } while (!count.compareAndSet(current, next));  // CAS更新
        return next;
    }
}
```

### 执行过程详解

```
线程A（收银员A）：
┌────────────────────────────────────┐
│ current = count.get();  // 100     │
│ next = 100 + 1;  // 101            │
│ compareAndSet(100, 101)            │
│   → 当前值=100，匹配！设为101      │
│   → 返回true，循环结束             │
└────────────────────────────────────┘

线程B（收银员B）：
┌────────────────────────────────────┐
│ current = count.get();  // 100     │
│ next = 100 + 1;  // 101            │
│ compareAndSet(100, 101)            │
│   → 当前值=101，不匹配！失败        │
│   → 返回false，继续循环            │
│                                    │
│ 重试：                             │
│ current = count.get();  // 101     │
│ next = 101 + 1;  // 102           │
│ compareAndSet(101, 102)            │
│   → 当前值=101，匹配！设为102      │
│   → 返回true，循环结束             │
└────────────────────────────────────┘
```

### 常用原子类

| 类名                           | 用途               |
| ---------------------------- | ---------------- |
| `AtomicInteger`              | 整数原子操作           |
| `AtomicLong`                 | 长整数原子操作          |
| `AtomicBoolean`              | 布尔值原子操作          |
| `AtomicReference<T>`         | 引用类型原子操作         |
| `AtomicStampedReference<T>`  | 带版本戳的引用（解决ABA问题） |
| `AtomicMarkableReference<T>` | 带标记的引用（变种版本号）    |

## CAS的三大问题

### 1. ABA问题

#### 问题描述

```
时间线：
T1: 线程1读取值=A
T2: 线程2将A改为B
T3: 线程2将B改回A
T4: 线程1的CAS检查：值还是A，通过！
```

**关键点**：线程1不知道值在中间被改过。

#### 瑞幸场景示例

**场景：用户取消订单后重新下单**

```
初始：用户优惠券张数 = 5张

T1: 用户A打开APP，读取优惠券数=5
T2: 用户A点击"取消订单"，系统扣减优惠券，恢复1张（5+1=6）
T3: 取消失败，系统回滚，优惠券恢复为5张
T4: 用户A点击"确认下单"，系统尝试扣减优惠券
    compareAndSet(5, 4) → 成功！
    
问题：用户A以为优惠券没动过，但中间经历了"扣减又恢复"
如果优惠券有有效期限制或其他业务规则，可能导致逻辑错误
```

### 2. 循环开销问题

#### 问题描述

CAS在竞争激烈时，可能导致**长时间自旋**，浪费CPU资源。

```java
// 高竞争场景示例
public int increment() {
    int current;
    int next;
    do {
        current = count.get();  // 每次重试都要重新读取
        next = current + 1;
    } while (!count.compareAndSet(current, next));  // 竞争激烈时频繁失败
    return next;
}
```

**解决思路**：

- 设置最大重试次数
- 使用`LongAdder`分段计数
- 退化为锁机制

### 3. 只能保证单个变量原子性

#### 问题描述

CAS只能保证**一个变量**的原子性，无法直接操作多个变量。

```java
// 错误的复合操作
public void transfer(Account from, Account to, int amount) {
    // 这不是原子操作！
    from.setBalance(from.getBalance() - amount);
    to.setBalance(to.getBalance() + amount);
}
```

**解决思路**：

- 使用`synchronized`
- 使用`AtomicReference`包装多个字段
- 使用数据库事务

## ABA问题解决方案

### 方案一：AtomicStampedReference（推荐）

#### 核心思想

在比较值的同时比较**版本戳**，版本戳单调递增。

```java
import java.util.concurrent.atomic.AtomicStampedReference;

public class LuckyCoffeeStock {
    // 库存：值 + 版本戳
    private AtomicStampedReference<Integer> stock = 
        new AtomicStampedReference<>(100, 0);
    
    public boolean deductStock(int quantity) {
        while (true) {
            // 获取当前快照（值 + 版本）
            Integer currentStock = stock.getReference();
            int currentStamp = stock.getStamp();
            
            if (currentStock < quantity) {
                return false;
            }
            
            // CAS更新：同时检查值和版本！
            boolean success = stock.compareAndSet(
                currentStock,          // 期望值
                currentStock - quantity, // 新值
                currentStamp,           // 期望版本
                currentStamp + 1        // 新版本
            );
            
            if (success) {
                return true;
            }
        }
    }
}
```

#### 瑞幸场景验证

```
初始：库存=100，版本=0

线程1：读取 库存=100，版本=0，准备扣减10
线程2：修改 库存=90，版本=1（用户下单）
线程2：修改 库存=100，版本=2（用户取消）

线程1：尝试 CAS(100, 90, 0, 1)
       → 当前版本=2，期望版本=0
       → 失败！版本不匹配

结果：线程1知道数据被修改过，需要重新读取
```

### 方案二：AtomicMarkableReference

适用于只需要"是否被修改过"的场景，比版本号开销更小。

```java
import java.util.concurrent.atomic.AtomicMarkableReference;

public class CacheEntry {
    private AtomicMarkableReference<String> value = 
        new AtomicMarkableReference<>("cached_data", false);
    
    public void update(String newValue) {
        while (true) {
            String[] current = new String[1];
            boolean[] marked = new boolean[1];
            value.get(current, marked);  // 获取当前值和标记
            
            // 如果已被标记删除，不更新
            if (marked[0]) {
                return;
            }
            
            if (value.compareAndSet(
                current[0], newValue,  // 值
                false, true              // 标记：false→true
            )) {
                return;
            }
        }
    }
}
```

### 方案三：数据库版本号（外部系统）

```sql
-- 结合CAS和数据库
UPDATE stock 
SET count = count - 1, version = version + 1 
WHERE id = 1 AND version = ?;

-- 检查影响行数
-- 如果 rowCount = 0，说明版本冲突，需要重试
```

## 性能对比

### CAS vs 锁

| 指标    | CAS       | synchronized |
| ----- | --------- | ------------ |
| 并发度   | 高（无阻塞）    | 低（阻塞等待）      |
| CPU开销 | 自旋消耗      | 上下文切换        |
| 适用场景  | 低竞争       | 高竞争          |
| 编程难度  | 较高（需处理重试） | 简单           |
| 公平性   | 非公平       | 可设置为公平       |

### AtomicInteger vs LongAdder（高竞争场景）

```java
// AtomicInteger：高竞争时性能急剧下降
public class Counter1 {
    private AtomicInteger count = new AtomicInteger(0);
    public void increment() {
        count.incrementAndGet();  // 竞争激烈时频繁失败
    }
}

// LongAdder：分段计数，显著提升高并发性能
public class Counter2 {
    private LongAdder count = new LongAdder();
    public void increment() {
        count.increment();  // 分段存储，减少竞争
    }
    public long get() {
        return count.sum();
    }
}
```

## 使用建议

### 适合使用CAS的场景

- ✅ 计数器、序列号生成
- ✅ 缓存更新
- ✅ 配置参数更新
- ✅ 低到中等竞争环境

### 不适合使用CAS的场景

- ❌ 复合操作（需要原子更新多个字段）
- ❌ 高竞争场景（循环开销大）
- ❌ 需要严格顺序保证

### 最佳实践

```java
public class BestPractice {
    private final AtomicStampedReference<Integer> stock = 
        new AtomicStampedReference<>(0, 0);
    
    private static final int MAX_RETRIES = 10;
    
    public boolean deductStock(int quantity) {
        for (int i = 0; i < MAX_RETRIES; i++) {
            Integer current = stock.getReference();
            int stamp = stock.getStamp();
            
            if (current < quantity) {
                return false;
            }
            
            if (stock.compareAndSet(current, current - quantity, stamp, stamp + 1)) {
                return true;
            }
            
            // 退避策略：避免过度CPU消耗
            Thread.onSpinWait();
        }
        
        // 超过最大重试次数，返回失败或降级
        return false;
    }
}
```

## 与其他并发机制对比

### 完整对比表

| 特性   | CAS    | synchronized | ReentrantLock |
| ---- | ------ | ------------ | ------------- |
| 原理   | 硬件原子指令 | JVM内置锁       | AQS框架         |
| 性能   | 低竞争最优  | 高竞争稳定        | 可中断、可超时       |
| 公平性  | 非公平    | 可配置          | 可配置           |
| 死锁风险 | 无      | 有            | 有（可避免）        |
| 重量级  | 轻量级    | 中等           | 可重入           |

### 选择指南

```
选择策略：
┌─────────────────────────────────┐
│ 竞争程度                        │
├─────────────────────────────────┤
│ 低竞争 → CAS（AtomicInteger）   │
│ 中等竞争 → LongAdder            │
│ 高竞争 → ReentrantLock          │
│ 复杂同步 → synchronized         │
└─────────────────────────────────┘
```

## 总结

### CAS的核心价值

1. **无锁化**：避免线程阻塞和上下文切换
2. **高性能**：硬件级原子操作，开销小
3. **乐观并发**：先尝试，失败再重试

### CAS的局限性

1. **ABA问题**：需要版本号机制解决
2. **循环开销**：高竞争时CPU消耗大
3. **单变量限制**：复合操作需其他机制

### 最佳实践

- 优先使用`AtomicStampedReference`解决ABA问题
- 设置最大重试次数，防止无限循环
- 高竞争场景考虑`LongAdder`或锁机制
- 复合操作使用`synchronized`或数据库事务

***

## 相关链接

- [[乐观锁]] - CAS是乐观锁的一种实现方式
- [[悲观锁]] - 悲观并发控制方式
- [[MVCC-多版本并发控制]] - 另一种并发控制机制
- [[隔离级别]] - 四种隔离级别详解
- [[锁机制实现详解]] - 数据库锁机制原理

