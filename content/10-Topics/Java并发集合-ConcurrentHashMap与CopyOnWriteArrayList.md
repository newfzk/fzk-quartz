---
title: 并发容器选型 - ConcurrentHashMap vs CopyOnWriteArrayList
date: 2026-05-26
updated: 2026-05-26 15:30:00
aliases:
  - Java并发容器
  - 并发集合
related:
  - "[[Java并发集合-ConcurrentHashMap]]"
  - "[[Java并发集合-CopyOnWriteArrayList]]"
  - "[[CAS-Compare-And-Swap]]"
  - "[[synchronized机制详解]]"
  - "[[乐观锁]]"
tags:
  - language/java
  - topic/java/集合
  - topic/java/并发
status: to-review
---

# 并发容器选型 — ConcurrentHashMap vs CopyOnWriteArrayList

本文是 [[Java并发集合-ConcurrentHashMap]] 与 [[Java并发集合-CopyOnWriteArrayList]] 的**对比与选型指南**。两张笔记分别深入讨论各自的原理和实现，本文聚焦于"什么时候用哪个"。

---

## 常见集合线程安全性

| 集合类型 | 线程安全 | 问题描述 |
|---------|---------|---------|
| `HashMap` | ❌ 不安全 | 多线程下可能死循环（JDK 1.7）、数据丢失 |
| `Hashtable` | ✅ 安全 | 方法级 synchronized，性能差 |
| `Collections.synchronizedMap` | ✅ 安全 | 包装器模式，全局锁，性能差 |
| `ConcurrentHashMap` | ✅ 安全 | 细粒度锁，读无锁，高性能 |

### HashMap 并发问题演示

```java
// 危险：多线程下HashMap可能死循环（JDK 1.7）
public class HashMapThreadIssue {
    private static HashMap<String, Integer> map = new HashMap<>();
    
    public static void main(String[] args) throws InterruptedException {
        for (int i = 0; i < 1000; i++) {
            final int num = i;
            new Thread(() -> {
                map.put("key" + num, num);
            }).start();
        }
        Thread.sleep(1000);
        System.out.println("Map size: " + map.size());
    }
}
```

**问题原因**：JDK 1.7 HashMap 扩容时链表反转可能形成环形链表，导致死循环。

---

## ConcurrentHashMap vs CopyOnWriteArrayList

| 特性 | ConcurrentHashMap | CopyOnWriteArrayList |
|------|------------------|---------------------|
| 数据结构 | 哈希表 | 动态数组 |
| 写操作开销 | 低（仅锁单个桶） | 高（复制整个数组） |
| 读操作开销 | 低（无锁） | 低（无锁） |
| 内存占用 | 适中 | 高（写时双倍内存） |
| 迭代一致性 | 弱一致性 | 快照一致性 |

---

## 选型指南

```
数据访问模式 → 选择策略
┌─────────────────────────────────────────────────────────┐
│ 读多写少 + 遍历频繁 → CopyOnWriteArrayList             │
│ 读多写少 + 随机访问 → ConcurrentHashMap                │
│ 写操作频繁         → ConcurrentHashMap（优于CopyOnWrite）│
│ 需要遍历一致性     → CopyOnWriteArrayList              │
└─────────────────────────────────────────────────────────┘
```

### 选择流程

```
开始选型：
  │
  ├─ 写操作频繁？──────────── 是 ─→ ConcurrentHashMap
  │
  ├─ 需要遍历时一致性快照？─── 是 ─→ CopyOnWriteArrayList
  │                                   （即使写少，也要有快照隔离）
  │
  ├─ 数据量预计很大？──────── 是 ─→ ConcurrentHashMap
  │                                   （COW 写时复制开销大）
  │
  ├─ 读多写少 + 随机访问？─── 是 ─→ ConcurrentHashMap
  │
  └─ 读多写少 + 遍历为主？─── 是 ─→ CopyOnWriteArrayList
```

---

## 本地缓存方案对比

| 方案 | 过期策略 | 最大容量 | 并发安全 | 适用场景 |
|------|---------|---------|---------|---------|
| `ConcurrentHashMap` | 手动实现 | 无限制 | ✅ | 简单缓存 |
| Caffeine | LRU/LFU/TTL | 支持 | ✅ | 高性能缓存 |
| Guava Cache | LRU | 支持 | ✅ | 通用缓存 |

> 对于复杂缓存需求，推荐使用 Caffeine 框架而非直接使用 `ConcurrentHashMap`，可参考 [[Java并发集合-ConcurrentHashMap]] 中的实战示例。

---

## 总结

### 核心要点

1. **[[Java并发集合-ConcurrentHashMap|ConcurrentHashMap]]**：JDK 1.8+ 使用 CAS + synchronized 实现细粒度锁，读操作无锁，适合高并发读写场景
2. **[[Java并发集合-CopyOnWriteArrayList|CopyOnWriteArrayList]]**：写时复制策略，读操作完全无锁，适合写极少、遍历频繁的场景（如配置列表）
3. **缓存优化**：配合 Caffeine 或 Guava Cache 设置过期时间和最大容量，避免内存无限增长
4. **并发安全**：普通 HashMap 多线程下不安全，必须使用并发容器或同步包装器

### 最佳实践

- 商品列表优先使用 `ConcurrentHashMap` 作为缓存容器
- 配置类数据使用 `CopyOnWriteArrayList`
- 配合本地缓存框架（Caffeine）实现自动过期策略
- 遍历集合时返回快照（如 `new ArrayList<>(cache.values())`）

---

## 参考链接

- [[Java并发集合-ConcurrentHashMap]] - ConcurrentHashMap 原理详解
- [[Java并发集合-CopyOnWriteArrayList]] - CopyOnWriteArrayList 原理详解
- [[CAS-Compare-And-Swap]] - CAS 机制详解
- [[synchronized机制详解]] - synchronized 同步机制
- [[乐观锁]] - 乐观并发控制方式
- [[红黑树面试常考知识点]] - 红黑树在 CHM 中的应用
