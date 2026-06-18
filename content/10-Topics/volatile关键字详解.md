---
title: volatile 关键字详解
date: 2026-05-28
aliases:
  - volatile
  - volatile关键字
  - 内存可见性
related:
  - "[[CAS-Compare-And-Swap]]"
  - "[[synchronized机制详解]]"
  - "[[Java并发集合-ConcurrentHashMap]]"
  - "[[transient关键字详解]]"
tags:
  - language/java
  - topic/java/并发
status: to-review
---

# `volatile` 关键字

`volatile` 属于 **Java 并发编程**关键字，解决**内存可见性**问题。

---

## 作用

1. **可见性**：`volatile` 变量的读写直接操作**主内存**，不走 CPU 缓存。一个线程修改后，另一线程**立即可见**
2. **禁止指令重排序**：编译器和 CPU 不会对 `volatile` 变量前后的指令进行重排序（通过**内存屏障**实现）

### 语义示意

```
普通变量：
┌──────┐     ┌──────────┐     ┌──────────┐     ┌──────┐
│Thread1│ ←→ │CPU缓存   │ ←→ │ 主内存    │ ←→ │CPU缓存│ ←→ │Thread2│
└──────┘     └──────────┘     └──────────┘     └──────┘
                                    ↑
修改对其他线程不可见（可能还在缓存中）

volatile 变量：
         ┌──────────┐
         │ 主内存    │ ← 直接读写主内存
         └──────────┘
         ↗         ↖
    ┌──────┐     ┌──────┐
    │Thread1│     │Thread2│
    └──────┘     └──────┘
    修改对其他线程立即可见
```

## 典型应用场景

```java
// 1. 状态标志位
private volatile boolean running = true;
public void stop() { running = false; }
// 线程检查 running，立即可见停止信号

// 2. Double-Checked Locking（单例模式）
private static volatile Singleton instance;
public static Singleton getInstance() {
    if (instance == null) {              // 第一重检查
        synchronized (Singleton.class) {
            if (instance == null) {      // 第二重检查（DCL）
                instance = new Singleton();
                // volatile 防止：分配空间 → 返回引用 → 初始化       ← 重排序导致问题
                //              必须：    分配空间 → 初始化 → 返回引用
            }
        }
    }
    return instance;
}
```

## 在 ConcurrentHashMap 中的体现

`ConcurrentHashMap` 大量使用 `volatile` 保证无锁读的可见性（参见 [[Java并发集合-ConcurrentHashMap]]）：

```java
public class ConcurrentHashMap<K, V> {
    transient volatile Node<K,V>[] table;    // 核心数组，volatile保证线程可见
    private transient volatile Node<K,V>[] nextTable;  // 扩容数组
    private transient volatile int sizeCtl;   // 控制标识符
}
```

当写线程修改了 `table` 引用（如扩容替换为新数组），读线程通过 `volatile` 语义**立即看到**最新值，无需加锁。

## 与 `synchronized` 对比

| 特性 | volatile | synchronized |
|------|----------|-------------|
| 可见性 | ✅ 保证 | ✅ 保证 |
| 原子性 | ❌ 不保证 | ✅ 保证（互斥） |
| 阻塞 | 无阻塞 | 可能阻塞线程 |
| 适用场景 | 单一变量的可见性 | 复合操作的原子性 |

## 与 `transient` 的区分

参见 [[transient关键字详解]]。两者在源码中常同时出现（如 ConcurrentHashMap），但职责完全不同：
- **volatile = 线程可见**（多线程场景）
- **transient = 序列化忽略**（IO/存储场景）

## 参考链接

- [[Java并发集合-ConcurrentHashMap]] — volatile 在 CHM 中的应用
- [[Java并发集合-ConcurrentHashMap与CopyOnWriteArrayList]] — 并发容器对比与选型
- [[CAS-Compare-And-Swap]] — CAS 与 volatile 配合实现无锁并发
- [[synchronized机制详解]] — 与 volatile 的对比
- [[transient关键字详解]] — transient 关键字详解