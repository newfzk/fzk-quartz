---
title: Java 集合框架（MOC）
date: 2026-06-11
tags:
  - topic/Java基础
  - topic/集合
  - topic/MOC
  - language/java
aliases:
  - Java集合
  - Java Collections
related:
  - "[[Java集合-ArrayList与LinkedList底层原理]]"
  - "[[Java并发集合-ConcurrentHashMap]]"
  - "[[Java并发集合-CopyOnWriteArrayList]]"
  - "[[Java集合框架-HashSet]]"
  - "[[Java-Map-computeIfAbsent]]"
  - "[[Java-Map-merge方法]]"
---

# Java 集合框架 — 知识地图

Java 集合框架（Java Collections Framework, JCF）是 Java 最核心的基础库之一，提供了**数据结构**的标准实现。本 MOC 汇总了相关的原子笔记。

> [!abstract] 本笔记为 MOC（Map of Content）
> 点击下方链接跳转到对应的原子笔记。

## List 体系

| 笔记 | 简介 |
|------|------|
| [[Java集合-ArrayList与LinkedList底层原理]] | ArrayList（动态数组）vs LinkedList（双向链表）底层原理、性能对比与选型 |

## Map 体系

| 笔记 | 简介 |
|------|------|
| [[Java并发集合-ConcurrentHashMap]] | ConcurrentHashMap 分段锁/CAS + synchronized 实现，高并发 Map |
| [[Java-Map-computeIfAbsent]] | Map.computeIfAbsent 的用法、源码分析与性能优化 |
| [[Java-Map-merge方法]] | Map.merge 的签名逻辑与多行记录归约典型用法 |

## Set 体系

| 笔记 | 简介 |
|------|------|
| [[Java集合框架-HashSet]] | HashSet 基于 HashMap 实现，集合运算（差集、并集、交集） |

## 并发集合

| 笔记 | 简介 |
|------|------|
| [[Java并发集合-ConcurrentHashMap]] | 线程安全 HashMap，1.7 分段锁 → 1.8 CAS + synchronized |
| [[Java并发集合-CopyOnWriteArrayList]] | 写时复制策略，读多写少场景的线程安全 List |
| [[Java并发集合-ConcurrentHashMap与CopyOnWriteArrayList]] | 并发容器对比与选型指南 |

## 快速参考

### 线程安全集合速查

| 场景 | 推荐实现 | 替代方案 |
|------|---------|---------|
| 无并发读多写少 | `ArrayList` | `LinkedList` |
| 无并发 KV 存储 | `HashMap` | `TreeMap`、`LinkedHashMap` |
| 并发读多写少 | `CopyOnWriteArrayList` | `Collections.synchronizedList` |
| 并发 KV 存储 | `ConcurrentHashMap` | `Collections.synchronizedMap` |
| 并发有序 KV | `ConcurrentSkipListMap` | — |
| 无重复元素 | `HashSet` | `TreeSet`、`LinkedHashSet` |

## 参考链接

- [[宇树科技-Java面试题]] — 面试题文档（一、Java 核心基础，第1题）
