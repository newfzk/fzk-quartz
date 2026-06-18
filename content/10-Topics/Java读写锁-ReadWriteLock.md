---
title: Java 读写锁（ReadWriteLock）
date: 2026-05-23
aliases:
  - ReadWriteLock
  - 读写锁
  - ReentrantReadWriteLock
  - 共享锁
  - 排他锁
related:
  - "[[锁机制实现详解]]"
  - "[[synchronized机制详解]]"
  - "[[悲观锁]]"
tags:
  - language/java
  - topic/java/并发
status: to-review
---

# Java 读写锁（ReadWriteLock）

> **说明**：Java 的 `ReadWriteLock` 是 **JVM 级别的并发控制工具**，与数据库完全无关。它属于 `java.util.concurrent.locks` 包，用于在单进程内实现细粒度的读写分离控制。

## ReadWriteLock 接口概述

`ReadWriteLock` 是 Java 并发包提供的接口，定义了一对锁：
- **读锁（共享锁）**：多个线程可同时获取，适合读多写少场景
- **写锁（排他锁）**：同一时间只能被一个线程持有

**核心特性**：
- **读写分离**：读操作不阻塞其他读操作，提高并发读性能
- **写优先/读优先**：不同实现有不同的锁获取策略
- **可重入性**：`ReentrantReadWriteLock` 支持锁的重入

## Java 读写锁实现

```java
import java.util.concurrent.locks.ReadWriteLock;
import java.util.concurrent.locks.ReentrantReadWriteLock;

public class Cache {
    private final ReadWriteLock rwLock = new ReentrantReadWriteLock();
    private Map<String, Object> cache = new HashMap<>();
    
    public Object get(String key) {
        rwLock.readLock().lock();
        try {
            return cache.get(key);
        } finally {
            rwLock.readLock().unlock();
        }
    }
    
    public void put(String key, Object value) {
        rwLock.writeLock().lock();
        try {
            cache.put(key, value);
        } finally {
            rwLock.writeLock().unlock();
        }
    }
}
```

## 读写锁特性

| 场景 | 读锁 | 写锁 |
|------|------|------|
| 多个读锁 | 允许 | 阻塞 |
| 读锁+写锁 | 阻塞 | 阻塞 |
| 多个写锁 | 阻塞 | 阻塞 |

## 与其他锁机制对比

| 锁类型 | 实现方式 | 并发性能 | 适用场景 |
|--------|----------|----------|----------|
| 悲观锁 | SELECT FOR UPDATE | 低 | 写多读少、库存扣减 |
| 乐观锁 | 版本号/CAS | 高 | 读多写少、普通更新 |
| 读写锁 | ReadWriteLock | 中高 | 缓存读写分离 |
| 分布式锁 | Redis/ZK | 中 | 跨进程/跨机器 |

## 参考链接

- [[锁机制实现详解]] — 锁的分类体系总览
- [[synchronized机制详解]] — Java 内置锁机制
- [[悲观锁]] — 数据库悲观锁
- [[分布式锁实现]] — Redis/ZK 分布式锁
- [[快手电商-一面-19题总结]] — Q4 ReadWriteLock 使用场景