---
title: Java wait/notify 机制
date: 2026-06-15
aliases:
  - wait-notify
  - 线程间通信
  - 等待通知机制
related:
  - "[[synchronized机制详解]]"
  - "[[Java读写锁-ReadWriteLock]]"
tags:
  - language/java
  - topic/java/并发
status: to-review
---

# Java wait/notify 机制

> `wait()` / `notify()` / `notifyAll()` 是 Java 内置的**线程间协作机制**，必须在 `synchronized` 代码块或方法中调用。

## 核心机制

### 前提条件

- 调用 `wait()` / `notify()` 的线程必须持有对象的**监视器锁（Monitor）**
- 即必须在 `synchronized` 块/方法中调用
- 否则抛出 `IllegalMonitorStateException`

### 执行流程

```
线程A 获取锁 → 条件不满足 → wait()释放锁 → 进入 Waiting 状态
线程B 获取锁 → 条件满足 → notify() → 线程B释放锁
线程A 被唤醒 → 重新竞争锁 → 获得锁后继续执行
```

## wait() 后线程什么时候被唤醒？

| 唤醒方式 | 说明 | 被唤醒时机 |
|---------|------|-----------|
| **notify()** | 随机唤醒一个等待线程 | 无法控制唤醒哪个线程 |
| **notifyAll()** | 唤醒所有等待线程 | 所有线程都会被唤醒 |
| **interrupt()** | 中断等待线程 | 抛出 `InterruptedException` |
| **Spurious Wakeup** | 虚假唤醒 | 即使没有 notify，线程也可能被唤醒 |

### 虚假唤醒（Spurious Wakeup）

- 操作系统层面的现象：即使没有 notify 信号，阻塞在 `wait()` 上的线程也可能被唤醒
- **解决方案**：始终在循环中检查条件

```java
// 错误的写法
if (!condition) {
    wait(); // 被唤醒后不再检查条件
}

// 正确的写法 — 始终在循环中 wait
synchronized (lock) {
    while (!condition) {
        lock.wait(); // 唤醒后重新检查条件
    }
    // 执行业务逻辑
}
```

## notify() vs notifyAll()

| 对比维度 | notify() | notifyAll() |
|---------|----------|-------------|
| 唤醒数量 | 随机一个等待线程 | 所有等待线程 |
| 安全性 | 低 — 可能导致信号丢失 | 高 — 所有线程一起竞争 |
| 性能 | 好 — 减少上下文切换 | 差 — 多线程竞争锁 |
| 推荐使用 | 确信只需要唤醒一个线程 | 不确定时使用（更安全） |

### 信号丢失场景

```java
// 场景：缓冲区空，生产者 put，消费者 take
// 如果使用 notify()，且唤醒的是另一个生产者（而非消费者）
// 消费者继续等待 → 死等（信号丢失）
// 解决方案：使用 notifyAll()
```

## 唤醒后需要重新获取 Monitor

- **是的**，线程被唤醒后不能立即从 `wait()` 返回
- 线程从 **Waiting → Blocked** 状态，排队等待获取 monitor 锁
- 获取到锁后，从 `wait()` 返回继续执行
- 如果获取不到锁，线程阻塞在锁竞争上

## 与 sleep() 的区别

| 特性 | wait() | sleep() |
|------|--------|---------|
| 持有锁时调用 | **释放锁** | **不释放锁** |
| 调用条件 | 必须在 synchronized 块中 | 任意地方 |
| 唤醒方式 | notify / notifyAll / 超时 / 中断 | 超时 / 中断 |
| 所属类 | Object | Thread |
| 作用 | 线程间协作（条件等待） | 暂停当前线程执行 |

## 参考链接

- [[快手电商-一面-19题总结]] — Q6 wait/notify 机制
- [[synchronized机制详解]] — 内置锁与 Monitor
- [[Java读写锁-ReadWriteLock]] — 显式锁机制
