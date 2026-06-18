---
title: Go-map实现集合
date: 2026-06-02
updated: 2026-06-02 00:00:00
tags:
  - language/go
  - topic/数据结构
status: to-review
---

## 核心概念

Go 语言没有内置 Set 类型，常规做法是使用 `map[K]struct{}` 或 `map[K]bool` 来模拟集合。

## 两种实现方式

| 方式 | 声明 | 添加 | 检查存在 | 优点 |
|------|------|------|----------|------|
| `map[K]bool` | `set := make(map[string]bool)` | `set[k] = true` | `if set[k]` | 直观、易读 |
| `map[K]struct{}` | `set := make(map[string]struct{})` | `set[k] = struct{}{}` | `_, ok := set[k]` | 零内存占用（struct{} 占 0 字节） |

## 对称差实现

```go
for k := range setA {
    if !setB[k] {
        fmt.Println("只在 a 中:", k)
    }
}
for k := range setB {
    if !setA[k] {
        fmt.Println("只在 b 中:", k)
    }
}
```

## 面试要点

- 面试推荐用 `map[string]bool`（更直观），讲原理时提到 `map[string]struct{}`（性能更优）
- Go 1.21+ 有实验性的 `slices`/`maps` 包但没有 Set，第三方用 `golang-set`
- `struct{}` 是 Go 的空结构体，宽度为 0，不占内存

## 参考链接

- [[柠檬微趣-笔试-Q1-文本文件去重|柠檬微趣-笔试-Q1-文本文件去重]]