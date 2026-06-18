---
title: HotSpot — Java 虚拟机实现
date: 2026-06-17
aliases:
  - HotSpot
  - HotSpot VM
  - Java HotSpot
  - OopMap
related:
  - "[[GC-Roots详解]]"
  - "[[JVM-堆内存分代模型]]"
  - "[[JVM-GC类型对比]]"
  - "[[JVM-对象创建与内存分配]]"
  - "[[JVM-OOM排查指南]]"
  - "[[synchronized机制详解]]"
tags:
  - language/java
  - topic/JVM
status: to-review
---

# HotSpot — Java 虚拟机实现

> HotSpot 是 **Oracle JDK / OpenJDK 默认的 Java 虚拟机实现**，也是目前最主流的 JVM，几乎所有 Java 程序员每天运行的程序都在它之上执行。

## 名字的由来：热点代码检测

HotSpot 的名字源于其核心特性——**热点代码检测（Hot Spot Detection）**：

```
字节码（慢） ──频繁执行──→ 热点代码 ──JIT编译──→ 本地机器码（快）
```

- 运行时持续监控代码执行频率
- 自动识别 **"热点"（Hot Spot）**——被频繁调用的方法或循环
- 将热点代码从解释执行**即时编译（JIT, Just-In-Time）** 为本地机器码，大幅提升性能
- 非热点代码仍保持解释执行，避免不必要的编译开销

> 这种"运行时分析 + 选择性编译"的策略，使得 HotSpot 在启动速度（无需提前编译所有代码）和长期运行性能（热点代码被优化到极致）之间取得平衡。

## HotSpot 在 GC 中的角色

[[GC-Roots详解|GC Roots 详解]] 中提到的那句话：

> HotSpot 使用 **OopMap** 记录栈帧中哪些位置是对象引用，精确引导 GC 遍历，而不是扫描栈上所有数据。

这里的上下文是：

### OopMap（Ordinary Object Pointer Map）

- **问题**：GC 进行可达性分析时，需要知道栈上哪些位置存的是对象引用
- **朴素方案**：逐字节扫描栈内存，判断每个值是否像指针（保守式 GC，不准确且可能误判）
- **HotSpot 的方案**：在**安全点（Safepoint）** 生成 OopMap，精确记录栈帧中哪些偏移量是对象引用
- **好处**：GC 可以**精确（Precise）遍历**，不遗漏、不误判

### HotSpot 的 GC 演进

| 阶段 | 垃圾回收器 | 特点 |
|------|-----------|------|
| 早期 | Serial / Parallel | 分代收集，串行或并行 |
| JDK 7+ | CMS（已弃用） | 追求低停顿 |
| JDK 9+ | G1（默认） | 区域化堆，可预测停顿 |
| JDK 11+ | ZGC | 几乎无停顿，大堆适用 |
| JDK 12+ | Shenandoah | 并发压缩，低延迟 |

## HotSpot 的其他关键特性

- **栈上分配与标量替换**：小对象可能直接在栈上分配（逃逸分析后），减轻 GC 压力
- **锁优化**：[[synchronized机制详解|synchronized]] 的偏向锁 → 轻量级锁 → 重量级锁的升级过程，是 HotSpot 特有的优化
- **分代收集**：新生代（Eden / Survivor）+ 老年代的堆结构（[[JVM-堆内存分代模型]]）
- **类加载机制**：双亲委派模型、自定义 ClassLoader

## 与其他 JVM 实现的对比

| JVM | 维护方 | 特点 |
|-----|-------|------|
| **HotSpot** | Oracle | 主流 JVM，OpenJDK 默认，功能最全 |
| **JRockit** | 原 BEA → Oracle | 曾被合并，其诊断和 Mission Control 特性融入 HotSpot |
| **GraalVM** | Oracle Labs | 高性能多语言 JVM，支持 AOT 编译、Truffle 框架 |
| **Zing** | Azul | 低延迟 GC（C4），无需停顿，适合金融交易等场景 |
| **OpenJ9** | Eclipse Foundation / IBM | 低内存占用，适合容器环境 |

## 一句话总结

> **HotSpot = 你电脑上运行 Java 程序的那个 JVM 的名字。** 在 GC 的语境下，讨论的就是 HotSpot JVM 是如何实现垃圾回收的（OopMap、分代收集、G1、ZGC 等特性都是它的一部分）。

## 参考资料

- [[GC-Roots详解]] — GC Roots 与 OopMap 的关系
- [[JVM-堆内存分代模型]] — 堆内存结构与分代
- [[JVM-GC类型对比]] — 各 GC 回收器对比
- [[JVM-对象创建与内存分配]] — 对象在 HotSpot 中的分配流程
- [[synchronized机制详解]] — HotSpot 的锁升级机制
