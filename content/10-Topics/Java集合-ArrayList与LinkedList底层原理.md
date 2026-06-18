---
title: Java集合-ArrayList与LinkedList底层原理
date: 2026-06-11
updated: 2026-06-11 00:00:00
tags:
  - language/java
  - topic/java/集合
status: to-review
---

## 核心概念

`ArrayList` 和 `LinkedList` 是 Java 集合框架中两种最常用的 `List` 实现。**ArrayList 基于动态数组**，擅长随机访问；**LinkedList 基于双向链表**，擅长头尾操作。两者都不是线程安全的。

## 底层实现对比

[[ArrayList—动态数组]]

[[LinkedList—双向链表]]

## 性能对比

| 操作 | ArrayList | LinkedList |
|------|-----------|------------|
| `get(i)` 随机访问 | **O(1)** ✅ 下标直取 | O(n) ❌ 链式遍历 |
| `add(E)` 尾部插入 | O(1) 均摊 ✅（扩容时 O(n)） | **O(1)** ✅ |
| `add(i, E)` 中间插入 | O(n) 批量移动 | O(n) 寻址 + O(1) 改指针 |
| `remove(i)` 删除 | O(n) 批量移动 | O(n) 寻址 + O(1) 改指针 |
| `addFirst/addLast` | O(n) 需要移动 | **O(1)** ✅ |
| `removeFirst/removeLast` | O(n) 需要移动 | **O(1)** ✅ |
| 内存占用 | **低** ✅ 仅存数据 | 高 ❌ 额外 2 引用/节点 |
| CPU 缓存 | **友好** ✅ 连续预读 | 不友好 ❌ 随机跳跃 |

## 扩容机制详解（ArrayList）

```
触发条件：size + 1 > elementData.length

流程：
1. 计算新容量 = 旧容量 + (旧容量 >> 1) （详见 [[Java-移位运算符#常见应用场景|移位运算符详解]]）
   e.g. 10 → 15 → 22 → 33 → 49 → 73 → 109 → ...
2. 创建新数组：new Object[newCapacity]
3. 拷贝元素：System.arraycopy(旧数组, 0, 新数组, 0, size)
4. 替换引用：elementData = 新数组
5. 插入新元素

优化手段：new ArrayList<>(expectedSize) 避免多次扩容
```

## 常见误区澄清

### 误区1：LinkedList 中间插入一定比 ArrayList 快
**事实**：LinkedList 中间插入需要先 O(n) 找到位置，再 O(1) 改指针。ArrayList 中间插入是 O(n) 批量移动。两者整体都是 O(n)，但 ArrayList 的连续内存拷贝（`System.arraycopy` 是 JVM 内在优化，接近 memcpy）往往比 LinkedList 的离散节点遍历**更快**。

### 误区2：LinkedList 适合所有增删频繁的场景
**事实**：LinkedList 只在**头尾增删**有优势（O(1)），中间增删因寻址开销和缓存不友好，大数据量下性能反而不如 ArrayList。

### 误区3：ArrayList 的 get 操作总是 O(1)
**事实**：对 `Object[]` 的 `get` 确实是 O(1)，但如果存储的是**引用类型**，需要两次内存访问（先取引用地址，再取对象）。不过相比 LinkedList，这仍然是巨大的性能优势。

## 面试要点

1. **扩容的核心参数**：初始容量 10、扩容因子 1.5（`old + (old >> 1)`）、均摊 O(1)
2. **LinkedList 为何不适合随机访问**：O(n) 链式遍历，且 cache miss 严重
3. **实际开发选型原则**：95% 用 ArrayList，明确需频繁头尾操作才用 LinkedList
4. **替代方案**：队列推荐 `ArrayDeque`（循环数组，性能优于 LinkedList），并发场景用 `CopyOnWriteArrayList`
5. **LinkedList 的历史变化**：JDK 1.6 前是**双向循环链表**（头节点的 prev 指向尾），JDK 1.7+ 改为**双向非循环链表**

## 参考链接

- [[宇树科技-Java面试题]] - 面试题文档（一、Java 核心基础，第1题）
