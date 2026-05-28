---
title: CopyOnWriteArrayList 原理与实现
date: 2026-05-26
tags:
  - topic/并发控制
  - topic/Java集合
  - language/java
status: to-review
updated: 2026-05-26 15:30:00
aliases:
  - CopyOnWriteArrayList
  - COW原理
  - 写时复制列表
related:
  - "[[Java并发集合-ConcurrentHashMap]]"
  - "[[锁机制实现详解]]"
  - "[[synchronized机制详解]]"
---

# CopyOnWriteArrayList 原理与实现

## 核心原理

`CopyOnWriteArrayList` 采用 **写时复制** 策略，读操作不加锁，写操作复制整个数组。

### 写时复制机制

```
CopyOnWriteArrayList 写操作流程：
┌─────────────────────────────────────────────────────────────┐
│                    初始状态                                  │
│  array = [A, B, C, D]  ← volatile引用                      │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    写操作（添加元素E）                        │
│  1. 加锁                                                    │
│  2. 复制原数组：newArray = [A, B, C, D]                     │
│  3. 新数组尾部添加：newArray = [A, B, C, D, E]              │
│  4. 原子替换引用：array = newArray                          │
│  5. 解锁                                                    │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    读操作（无锁）                            │
│  读取的始终是旧数组的快照，不阻塞写操作                       │
└─────────────────────────────────────────────────────────────┘
```

**核心优势**：读写分离，读操作完全无锁，适用于[[Java并发集合-ConcurrentHashMap|ConcurrentHashMap]]不适用的遍历一致性场景。

## 核心实现

### 添加操作

```java
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();  // 写操作必须加锁
    try {
        Object[] elements = getArray();
        int len = elements.length;
        
        // 复制整个数组（关键：写时复制）
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        newElements[len] = e;
        
        // 原子替换数组引用
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}
```

### 读取操作

```java
public E get(int index) {
    // 读操作完全无锁
    return get(getArray(), index);
}

private E get(Object[] a, int index) {
    return (E) a[index];
}
```

### 批量写入优化

```java
public boolean addAll(Collection<? extends E> c) {
    Object[] cs = c.toArray();
    if (cs.length == 0)
        return false;
    
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        
        // 只需复制一次，优化批量写入
        Object[] newElements = Arrays.copyOf(elements, len + cs.length);
        System.arraycopy(cs, 0, newElements, len, cs.length);
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}
```

## 适用场景

| 场景 | 推荐度 | 说明 |
|------|--------|------|
| 配置列表（写极少，读频繁） | ✅ 强烈推荐 | 读操作零开销 |
| 事件监听器列表 | ✅ 推荐 | 遍历期间可安全修改 |
| 商品分类/标签列表 | ✅ 推荐 | 数据相对稳定 |
| 频繁写入的场景 | ❌ 不推荐 | 每次写都复制数组，性能差 |

## 与其他写时复制技术对比

`CopyOnWriteArrayList` 的写时复制思想也可以应用于 `CopyOnWriteArraySet`（基于 COW 实现）。写时复制的核心权衡：

- **内存**：写操作期间同时持有新旧数组
- **一致性**：迭代器遍历的是创建时的快照，不会反映后续修改
- **适用**：读多写少、数据量可控的场景

## 相关链接

- [[Java并发集合-ConcurrentHashMap]] - 另一个并发容器，适用于高并发读写
- [[Java并发集合-ConcurrentHashMap与CopyOnWriteArrayList]] - 并发容器对比与选型
- [[锁机制实现详解]] - ReentrantLock 实现原理
- [[synchronized机制详解]] - synchronized 同步机制对比
