---
tags:
  - topic/并发读写
  - status/to-review
  - language/java
created: 2026-05-20
---
## 📌 问题描述

门店POS机和小程序同时刷新商品列表，如何用ConcurrentHashMap或CopyOnWriteArrayList实现缓存优化，避免并发读写问题？

## ✅ 回答要点
商品列表属于读多写少的数据，我会优先使用`ConcurrentHashMap`作为缓存容器，因为它采用分段锁/CAS机制，支持高并发读写且线程安全。对于列表遍历操作，可以返回`new ArrayList<>(cache.values())`快照。如果必须用`CopyOnWriteArrayList`，它适合读远多于写的场景（如配置列表），每次修改会复制整个数组，因此写操作少时可以用，但门店和小程序并发刷新（写操作）频繁时会有性能问题。我选择`ConcurrentHashMap`配合本地缓存过期策略。


## 🔗 相关知识

- **ConcurrentHashMap**：JDK1.8后使用CAS + synchronized，锁粒度细，读操作无锁，写操作只锁桶。
    
- **CopyOnWriteArrayList**：读操作不加锁，写操作复制新数组，适合写极少、遍历频繁的场景。
    
- **并发读写问题**：普通HashMap多线程下可能死循环、数据丢失；使用`Collections.synchronizedMap`性能差。
    
- **缓存优化**：可配合Caffeine或Guava Cache设置过期时间、最大容量，避免内存无限增长。

## 💡 记忆技巧


## 📚 参考资料
