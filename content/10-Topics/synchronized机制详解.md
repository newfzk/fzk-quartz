---
title: synchronized 机制详解
date: 2026-05-28
tags:
  - topic/并发控制
  - language/java
status: to-review
aliases:
  - synchronized
  - Java内置锁
  - 监视器锁
  - 管程
related:
  - "[[锁机制实现详解]]"
  - "[[CAS-Compare-And-Swap]]"
  - "[[乐观锁]]"
  - "[[悲观锁]]"
---

## 一、synchronized 是什么

`synchronized` 是 Java 提供的内置锁（Intrinsic Lock）机制，基于 **Monitor（管程/监视器）** 模型实现。它保证：

- **原子性**：被锁住的代码块在线程执行期间不会被其他线程中断
- **可见性**：释放锁前对变量的修改，对后续获取锁的线程立即可见
- **有序性**：禁止指令重排序，保证 happens-before 规则

> `synchronized` 是一种**悲观锁** —— 默认认为并发冲突会发生，因此主动加锁阻塞。这与 `CAS` 的乐观策略相反。

## 二、使用方式

### 2.1 同步方法

```java
// 修饰实例方法 — 锁是当前实例对象（this）
public synchronized void instanceMethod() {
    // 线程安全的操作
}

// 修饰静态方法 — 锁是当前类的 Class 对象
public static synchronized void staticMethod() {
    // 线程安全的操作
}
```

### 2.2 同步代码块

```java
// 锁指定对象
public void blockMethod() {
    synchronized (this) {
        // 临界区
    }
}

// 锁 Class 对象
public void classBlock() {
    synchronized (MyClass.class) {
        // 临界区
    }
}

// 锁任意对象
private final Object lock = new Object();
public void customLock() {
    synchronized (lock) {
        // 临界区
    }
}
```

### 2.3 与 ConcurrentHashMap 的关系

在 ConcurrentHashMap 的 `putVal` 方法中，`synchronized` 用来锁定**单个桶的头节点**：

```java
synchronized (f) {   // f 是桶的头节点
    if (tabAt(tab, i) == f) {  // 双重检查
        // 安全的链表/红黑树操作
    }
}
```

这是**细粒度锁**的典型应用 —— 只锁一个桶而非整个表，不同桶的写入可以并发执行。

## 三、底层实现原理

### 3.1 Monitor 模型

每个 Java 对象关联一个 Monitor 对象（C++ 实现的 `ObjectMonitor`）：

```
线程进入 synchronized 时：
┌──────────┐          ┌─────────────────┐
│  Thread A │─────────→│  ObjectMonitor  │
└──────────┘          │                 │
                      │  _owner     ←─── 当前持有锁的线程
┌──────────┐          │  _EntryList ←─── 等待获取锁的线程队列
│  Thread B │─────────→│  _WaitSet   ←─── 调用 wait() 的线程集合
└──────────┘          └─────────────────┘
```

ObjectMonitor 核心字段：

| 字段 | 作用 |
|------|------|
| `_owner` | 当前持有锁的线程 |
| `_EntryList` | 阻塞队列，存放等待获取锁的线程 |
| `_WaitSet` | 等待队列，存放调用 `wait()` 后释放锁的线程 |
| `_recursions` | 重入次数（synchronized 支持重入） |

### 3.2 对象头与 Mark Word

`synchronized` 的锁信息存储在对象的 **Mark Word** 中（对象头的一部分，64 位 JVM 占 8 字节）：

```
64位 JVM 的 Mark Word 布局（无锁状态）：
┌─────────────────────────────────────────────────────────────┐
│        unused:25     │   identity_hashcode:31   │  unused:1 │  age:4  │  biased_lock:1  │  lock:2  │
└─────────────────────────────────────────────────────────────┘

锁标志位含义：
lock=01, biased_lock=0 → 无锁
lock=01, biased_lock=1 → 偏向锁
lock=00               → 轻量级锁
lock=10               → 重量级锁
lock=11               → GC标记
```

## 四、锁升级过程（JDK 1.6+）

为了减少锁带来的性能开销，JDK 1.6 对 `synchronized` 做了大量优化。锁可以单向升级，**不可降级**：

```
无锁 → 偏向锁 → 轻量级锁 → 重量级锁
└────────── 锁升级（单向、不可逆）──────────→
```

### 4.1 偏向锁

**设计目标**：消除线程无竞争时的同步开销。

- 锁记录线程 ID，偏向第一个获取它的线程
- 同一线程再次进入时，无需 CAS 操作，直接执行
- **撤销条件**：其他线程尝试获取该锁时，偏向锁被撤销

```java
// 偏向锁延迟：JVM 启动后默认 4 秒后才启用偏向锁
// -XX:BiasedLockingStartupDelay=0 可关闭延迟
```

### 4.2 轻量级锁

**设计目标**：线程交替执行（无实际竞争）时，用 CAS 代替互斥量。

- 线程在栈帧中创建 **Lock Record（锁记录）**
- 用 CAS 尝试将 Mark Word 替换为指向 Lock Record 的指针
- **升级条件**：CAS 失败，说明存在竞争，升级为重量级锁

### 4.3 重量级锁

**设计目标**：多线程真正竞争时，阻塞未获取锁的线程。

- 依赖操作系统的互斥量（Mutex Lock）
- 未获取锁的线程进入 `_EntryList` 阻塞
- 涉及**用户态 ↔ 内核态**切换，开销最大

### 4.4 锁升级流程

```
线程尝试获取锁
     │
     ▼
┌────────────────┐
│  检查 Mark Word │
└────────┬───────┘
         │
    ┌────┴────┐
    │ 无竞争   │ ─────────→ 偏向锁（记录线程ID）→ 重入直接进入
    └────┬────┘
         │ 其他线程尝试获取
         ▼
    ┌───────────┐
    │ 轻微竞争   │ ─────────→ 轻量级锁（自旋+CAS）
    └─────┬─────┘
          │ 自旋失败/竞争加剧
          ▼
     ┌──────────┐
     │ 激烈竞争  │ ─────────→ 重量级锁（操作系统互斥量）
     └────┬─────┘
          │ 线程进入 _EntryList 阻塞
          ▼
      等待唤醒
```

## 五、JDK 1.6 优化

### 5.1 锁消除

JIT 编译器通过逃逸分析，发现对象不会被其他线程访问时，直接去掉 `synchronized`：

```java
public void method() {
    Object lock = new Object();  // 局部对象，不逃逸
    synchronized (lock) {
        // JIT 直接消除锁
    }
}
```

### 5.2 锁粗化

JIT 将相邻的多个同步块合并为一个，减少加解锁次数：

```java
// 优化前：多次加解锁
synchronized (this) { foo(); }
synchronized (this) { bar(); }
synchronized (this) { baz(); }

// 优化后：一次加解锁
synchronized (this) {
    foo();
    bar();
    baz();
}
```

### 5.3 自适应自旋

轻量级锁阶段，JVM 会根据上次自旋等待的成功率，动态调整自旋次数：

- 上次自旋成功 → 这次多自旋几次
- 上次自旋失败 → 这次少自旋或直接挂起

## 六、synchronized vs ReentrantLock

| 特性 | synchronized | ReentrantLock |
|------|-------------|---------------|
| 锁机制 | 内置锁（JVM 层面） | API 层面（`java.util.concurrent.locks`） |
| 使用方式 | 关键字，自动加解锁 | `lock()`/`unlock()`，需 finally 释放 |
| 性能 | JDK 1.6 优化后与 ReentrantLock 相近 | 相近 |
| 灵活性 | 低：不可中断，不可超时 | 高：可中断，可超时，可轮询 |
| 公平性 | 非公平（无法选择） | 支持公平和非公平 |
| 条件变量 | 配合 `wait()/notify()` | 支持多个 `Condition` |
| 锁状态 | 无法查询 | `tryLock()`、`isHeldByCurrentThread()` |

```java
// ReentrantLock 示例（需要手动解锁）
import java.util.concurrent.locks.ReentrantLock;

public class ReentrantLockExample {
    private final ReentrantLock lock = new ReentrantLock(true); // 公平锁
    
    public void doSomething() {
        lock.lock();
        try {
            // 临界区
        } finally {
            lock.unlock(); // 必须在 finally 中释放
        }
    }
    
    // 可超时获取锁
    public boolean tryDoSomething(long timeout, TimeUnit unit) 
            throws InterruptedException {
        if (lock.tryLock(timeout, unit)) {
            try {
                // 临界区
                return true;
            } finally {
                lock.unlock();
            }
        }
        return false; // 超时未获取到锁
    }
}
```

**选型建议**：
- 简单同步场景优先用 `synchronized`（代码更简洁，不易出错）
- 需要超时、可中断、多条件等待时用 `ReentrantLock`
- `ConcurrentHashMap` 选择 `synchronized` 而非 `ReentrantLock`，是因为 JDK 1.6 后两者性能相近，且 `synchronized` 语法更简洁，JVM 对其有更多优化空间

## 七、常见面试题

### Q1：synchronized 与 volatile 的区别？

| 特性 | synchronized | volatile |
|------|-------------|----------|
| 原子性 | ✅ 保证 | ❌ 不保证（仅保证单个读写） |
| 可见性 | ✅ 保证 | ✅ 保证 |
| 有序性 | ✅ 保证 | ✅ 禁止指令重排 |
| 阻塞 | ✅ 会阻塞 | ❌ 不阻塞 |
| 开销 | 较高 | 较低 |

### Q2：synchronized 是否会引发死锁？

会。两个线程互相等待对方释放锁时发生死锁：

```java
// 死锁示例
public class DeadlockDemo {
    private final Object lockA = new Object();
    private final Object lockB = new Object();
    
    public void method1() {
        synchronized (lockA) {
            synchronized (lockB) {
                // 操作
            }
        }
    }
    
    public void method2() {
        synchronized (lockB) {   // 与 method1 获取顺序相反
            synchronized (lockA) {  // → 可能死锁
                // 操作
            }
        }
    }
}
```

**避免死锁**：
1. 固定锁获取顺序（总是先 lockA 后 lockB）
2. 使用 `tryLock()` 超时机制（ReentrantLock）
3. 使用 `java.util.concurrent` 的高级工具类

### Q3：ConcurrentHashMap 为什么用 synchronized 而不是 ReentrantLock？

JDK 1.8 的 ConcurrentHashMap 使用 `synchronized` 锁桶头节点，主要考虑：

1. **性能相近**：JDK 1.6 优化后，synchronized 轻量级锁性能已不输 ReentrantLock
2. **语法简洁**：synchronized 自动释放锁，无需在 finally 中 unlock
3. **JVM 优化**：JIT 对 synchronized 有更多优化（锁消除、锁粗化等）
4. **历史简化**：JDK 1.7 的 Segment（继承 ReentrantLock）设计过于复杂，JDK 1.8 用 synchronized 简化实现

## 八、总结

```
synchronized 核心要点：
├── 是什么：Java 内置锁（Monitor 模型），保证原子性+可见性+有序性
├── 使用方式：修饰方法 / 修饰代码块
├── 对象信息：存储在对象头的 Mark Word 中
├── 锁升级：无锁 → 偏向锁 → 轻量级锁 → 重量级锁（单向不可逆）
├── JDK优化：锁消除 / 锁粗化 / 自适应自旋
└── 对比 ReentrantLock：简单场景用 synchronized，复杂场景用 ReentrantLock
```

## 参考链接

- [[CAS-Compare-And-Swap]] - CAS 机制详解
- [[锁机制实现详解]] - 锁的分类与对比
- [[乐观锁]] - 乐观锁 vs 悲观锁
- [[悲观锁]] - 悲观锁策略
- [[Java并发集合-ConcurrentHashMap]] - ConcurrentHashMap 中的 synchronized 应用
- [[Java并发集合-ConcurrentHashMap与CopyOnWriteArrayList]] - 并发容器对比与选型
