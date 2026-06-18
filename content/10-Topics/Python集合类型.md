---
title: Python集合类型
date: 2026-06-02
updated: 2026-06-02 00:00:00
tags:
  - language/python
  - topic/数据结构
status: to-review
---

## 核心概念

Python `set` 是基于哈希表的无序不重复集合，原生支持集合运算，适合去重、交集/差集/并集等操作。

## 集合运算

```python
only_a = set_a - set_b       # 差集 A - B
only_b = set_b - set_a       # 差集 B - A
only_one = set_a ^ set_b     # 对称差 A △ B
common = set_a & set_b       # 交集
all_ids = set_a | set_b      # 并集
```

| 运算符 | 方法 | 含义 |
|--------|------|------|
| `-` | `.difference()` | 差集 |
| `^` | `.symmetric_difference()` | 对称差 |
| `&` | `.intersection()` | 交集 |
| `\|` | `.union()` | 并集 |
| `<=` | `.issubset()` | 子集 |
| `>=` | `.issuperset()` | 超集 |

## 集合推导式

```python
set_a = {line.strip() for line in open('a.txt') if line.strip()}
```

## 面试要点

- `set` 内部基于哈希表，查找 O(1)，但比 `frozenset` 可修改
- set 运算符比方法调用更 Pythonic，面试推荐用 `-`、`&`、`|`、`^`
- 空集合用 `set()` 而不是 `{}`（`{}` 是空字典）

## 参考链接

- [[柠檬微趣-笔试-Q1-文本文件去重|柠檬微趣-笔试-Q1-文本文件去重]]