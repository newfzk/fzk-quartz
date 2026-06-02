---
title: Java关键字 - transient与volatile（MOC）
date: 2026-05-28
tags:
  - topic/Java基础
  - topic/并发控制
  - topic/MOC
  - language/java
aliases:
  - Java关键字
related:
  - "[[volatile关键字详解]]"
  - "[[transient关键字详解]]"
  - "[[Java并发集合-ConcurrentHashMap]]"
---

# `transient` 与 `volatile` — 知识地图

`transient` 和 `volatile` 是 Java 中两个**用途完全不同**的关键字，只是在源码（如 [[Java并发集合-ConcurrentHashMap]]）中经常**同时出现**。

> [!abstract] 本笔记为 MOC（Map of Content）
> 点击下方链接跳转到对应的原子笔记。

| 笔记 | 简介 |
|------|------|
| [[volatile关键字详解]] | 内存可见性、禁止指令重排序、DCL 单例模式 |
| [[transient关键字详解]] | 序列化忽略、敏感信息保护、自定义序列化 |

## 快速对比

| 特性 | `volatile` | `transient` |
|------|-----------|-------------|
| **所属领域** | 并发编程 | 序列化机制 |
| **核心作用** | 保证线程间可见性，禁止重排序 | 标记字段不参与序列化 |
| **记忆口诀** | volatile = 线程可见 | transient = 序列化忽略 |

## 参考链接

- [[Java并发集合-ConcurrentHashMap]] — volatile 和 transient 在 CHM 中的应用
- [[Java并发集合-ConcurrentHashMap与CopyOnWriteArrayList]] — 并发容器对比与选型
- [[CAS-Compare-And-Swap]] — CAS 与 volatile 配合实现无锁并发
- [[synchronized机制详解]] — 与 volatile 的对比