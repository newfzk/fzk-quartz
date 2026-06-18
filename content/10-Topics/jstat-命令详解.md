---
title: jstat 命令详解
type: basic-note
date: 2026-06-03
tags:
  - language/java
  - topic/JVM
  - topic/Linux/命令
status: to-review
---

# jstat 命令详解

`jstat` 是 JDK 自带的 JVM 监控工具，用于查看堆内存和 GC 状态。

## 基本用法

```shell
jstat -gc <pid> <间隔ms>
```

## 输出列含义

| 列名 | 全称 | 含义 |
|------|------|------|
| S0C | Survivor 0 Capacity | 幸存区0的总容量 |
| S1C | Survivor 1 Capacity | 幸存区1的总容量 |
| S0U | Survivor 0 Used | 幸存区0的已使用大小 |
| S1U | Survivor 1 Used | 幸存区1的已使用大小 |
| EC | Eden Capacity | 伊甸园区的总容量 |
| EU | Eden Used | 伊甸园区的已使用大小 |
| OC | Old Capacity | 老年代的总容量 |
| OU | Old Used | 老年代的已使用大小 |
| MC | Metaspace Capacity | 元空间的总容量（JDK8+） |
| MU | Metaspace Used | 元空间的已使用大小 |
| CCSC | Compressed Class Space Capacity | 压缩类空间的总容量 |
| CCSU | Compressed Class Space Used | 压缩类空间的已使用大小 |
| YGC | Young GC Count | 新生代 GC 总次数 |
| YGCT | Young GC Time | 新生代 GC 总耗时（秒） |
| FGC | Full GC Count | Full GC 总次数 |
| FGCT | Full GC Time | Full GC 总耗时（秒） |
| CGC | Concurrent GC Count | 并发 GC 次数（G1GC 特有） |
| CGCT | Concurrent GC Time | 并发 GC 总耗时（秒） |
| GCT | Total GC Time | 所有 GC 总耗时（秒） |

## 输出示例

```shell
$ jstat -gc 1 1000
 S0C    S1C     S0U    S1U      EC      EU       OC       OU      MC     MU    CCSC   CCSU   YGC  YGCT  FGC FGCT  CGC  CGCT    GCT
 0.0  29696.0   0.0  29696.0 307200.0 274432.0 627712.0 566141.0 222848.0 220615.3 22336.0 21113.1 214 6.156  0  0.000  78  0.747  6.903
```

## 关键指标解读

- **OU 接近 OC**：老年代快满了，即将触发 Full GC
- **FGC > 0**：已发生 Full GC，需关注
- **MU 接近 MC**：元空间不足，需扩容或排查类加载泄漏
- **YGC 单次耗时** = YGCT / YGC，超过 50ms 需关注

## 相关笔记

- [[JVM-堆内存分代模型]]
- [[JVM-OOM排查指南]]
- [[JVM内存溢出预防]]
- [[JVM-GC类型对比]]