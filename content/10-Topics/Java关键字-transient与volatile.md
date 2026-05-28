---
title: Java关键字 - transient与volatile
date: 2026-05-28
tags:
  - topic/Java基础
  - topic/并发控制
  - language/java
aliases:
  - transient
  - volatile
  - Java关键字
related:
  - "[[CAS-Compare-And-Swap]]"
  - "[[synchronized机制详解]]"
  - "[[Java并发集合-ConcurrentHashMap]]"
---

# `transient` 与 `volatile` 关键字

`transient` 和 `volatile` 是 Java 中两个**用途完全不同**的关键字，只是在源码（如 [[Java并发集合-ConcurrentHashMap]]）中经常**同时出现**。

---

## `volatile` — 可见性与禁止重排序

`volatile` 属于 **Java 并发编程**关键字，解决**内存可见性**问题。

### 作用

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

### 典型应用场景

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

### 在 ConcurrentHashMap 中的体现

`ConcurrentHashMap` 大量使用 `volatile` 保证无锁读的可见性（参见 [[Java并发集合-ConcurrentHashMap]]）：

```java
public class ConcurrentHashMap<K, V> {
    transient volatile Node<K,V>[] table;    // 核心数组，volatile保证线程可见
    private transient volatile Node<K,V>[] nextTable;  // 扩容数组
    private transient volatile int sizeCtl;   // 控制标识符
}
```

当写线程修改了 `table` 引用（如扩容替换为新数组），读线程通过 `volatile` 语义**立即看到**最新值，无需加锁。

### 与 `synchronized` 对比

| 特性 | volatile | synchronized |
|------|----------|-------------|
| 可见性 | ✅ 保证 | ✅ 保证 |
| 原子性 | ❌ 不保证 | ✅ 保证（互斥） |
| 阻塞 | 无阻塞 | 可能阻塞线程 |
| 适用场景 | 单一变量的可见性 | 复合操作的原子性 |

---

## `transient` — 序列化忽略

`transient` 属于 **Java 序列化机制**关键字，标记某字段**不参与序列化**。

### 作用

- 用 `transient` 修饰的字段，在对象序列化（写入文件 / 网络传输）时**被跳过**
- 反序列化时，`transient` 字段恢复为**默认值**（引用类型 → `null`，基本类型 → `0`/`false`）

### 典型应用场景

```java
public class User implements Serializable {
    private String username;
    private transient String password;       // 密码不序列化，防泄露
    private transient Logger logger;         // Logger 对象无需序列化
    private transient ThreadLocal<Context> ctx; // ThreadLocal 不可序列化
}
```

### 为什么需要 transient？

| 原因 | 说明 |
|------|------|
| **敏感信息** | 密码、密钥等不应写入持久化文件或网络传输 |
| **不可序列化对象** | `ThreadLocal`、`Socket`、`OutputStream` 等不具备序列化能力 |
| **可重建的派生数据** | 缓存结果、临时状态，反序列化后可重新计算 |
| **性能优化** | 减少序列化数据量，提升性能 |

### 在 ConcurrentHashMap 中的体现

```java
public class ConcurrentHashMap<K, V> {
    transient volatile Node<K,V>[] table;    // table 是运行时数据结构

    private transient volatile int sizeCtl;   // 只在内存中维护

    // 序列化时自定义逻辑：
    // ConcurrentHashMap 实现了 writeObject() 自定义序列化
    // 将键值对逐个写出，而非序列化内部复杂的 Node 结构
}
```

ConcurrentHashMap 内部的 `table` 数组用 `transient` 修饰，因为默认的 Java 序列化无法正确处理其内部的并发结构和链表/红黑树。ConcurrentHashMap 通过自定义 `writeObject()`/`readObject()` 方法，只序列化**键值对本身**，反序列化时再重新构建哈希表。

---

## 两者对比

| 特性 | `volatile` | `transient` |
|------|-----------|-------------|
| **所属领域** | 并发编程 | 序列化机制 |
| **核心作用** | 保证线程间可见性，禁止重排序 | 标记字段不参与序列化 |
| **修饰目标** | 成员变量 | 成员变量 |
| **影响范围** | 多线程读写行为 | 序列化/反序列化行为 |
| **同时出现原因** | 并发容器需要保证可见性，同时又需要控制序列化 | |

## 记忆要点

1. **volatile = 线程可见**（多线程场景联想）
2. **transient = 序列化忽略**（IO/存储场景联想）
3. 两者没有直接关系，只是常同时出现在并发集合的源码中

## 参考链接

- [[Java并发集合-ConcurrentHashMap]] — volatile 和 transient 在 CHM 中的应用
- [[Java并发集合-ConcurrentHashMap与CopyOnWriteArrayList]] — 并发容器对比与选型
- [[CAS-Compare-And-Swap]] — CAS 与 volatile 配合实现无锁并发
- [[synchronized机制详解]] — 与 volatile 的对比
