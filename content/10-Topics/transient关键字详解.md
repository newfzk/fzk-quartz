---
title: transient 关键字详解
date: 2026-05-28
aliases:
  - transient
  - transient关键字
  - 序列化忽略
related:
  - "[[volatile关键字详解]]"
  - "[[Java并发集合-ConcurrentHashMap]]"
tags:
  - language/java
  - topic/java/基础
  - topic/序列化
status: to-review
---

# `transient` 关键字

`transient` 属于 **Java 序列化机制**关键字，标记某字段**不参与序列化**。

---

## 作用

- 用 `transient` 修饰的字段，在对象序列化（写入文件 / 网络传输）时**被跳过**
- 反序列化时，`transient` 字段恢复为**默认值**（引用类型 → `null`，基本类型 → `0`/`false`）

## 典型应用场景

```java
public class User implements Serializable {
    private String username;
    private transient String password;       // 密码不序列化，防泄露
    private transient Logger logger;         // Logger 对象无需序列化
    private transient ThreadLocal<Context> ctx; // ThreadLocal 不可序列化
}
```

## 为什么需要 transient？

| 原因 | 说明 |
|------|------|
| **敏感信息** | 密码、密钥等不应写入持久化文件或网络传输 |
| **不可序列化对象** | `ThreadLocal`、`Socket`、`OutputStream` 等不具备序列化能力 |
| **可重建的派生数据** | 缓存结果、临时状态，反序列化后可重新计算 |
| **性能优化** | 减少序列化数据量，提升性能 |

## 在 ConcurrentHashMap 中的体现

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

## 与 `volatile` 的区分

参见 [[volatile关键字详解]]。两者在源码中常同时出现（如 ConcurrentHashMap），但职责完全不同：

| 特性 | `volatile` | `transient` |
|------|-----------|-------------|
| **所属领域** | 并发编程 | 序列化机制 |
| **核心作用** | 保证线程间可见性，禁止重排序 | 标记字段不参与序列化 |
| **修饰目标** | 成员变量 | 成员变量 |
| **影响范围** | 多线程读写行为 | 序列化/反序列化行为 |

## 记忆要点

1. **transient = 序列化忽略**（IO/存储场景联想）
2. **volatile = 线程可见**（多线程场景联想），参见 [[volatile关键字详解]]
3. 两者没有直接关系，只是常同时出现在并发集合的源码中

## 参考链接

- [[Java并发集合-ConcurrentHashMap]] — transient 和 volatile 在 CHM 中的应用
- [[volatile关键字详解]] — volatile 关键字详解
- [[Java并发集合-ConcurrentHashMap与CopyOnWriteArrayList]] — 并发容器对比与选型