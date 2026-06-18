---
title: JVM GC 类型对比
date: 2026-06-15
aliases:
  - 垃圾回收器对比
  - GC 选型
  - G1 vs ZGC vs Shenandoah
related:
  - "[[JVM-堆内存分代模型]]"
  - "[[JVM-对象创建与内存分配]]"
  - "[[GC-Roots详解]]"
  - "[[jstat-命令详解]]"
tags:
  - language/java
  - topic/JVM
status: to-review
---

# JVM GC 类型对比

> JVM 提供的 GC（Garbage Collection）按**回收范围**和**回收器实现**两个维度分类。理解不同 GC 类型的特点，是 JVM 调优和面试的核心。

---

## 一、按回收范围分类

JVM 中的 GC 事件按**发生区域**分为三类：

| GC 类型 | 回收范围 | 触发条件 | 特点 |
|---------|---------|---------|------|
| **Minor GC / YGC** | 新生代（Eden + Survivor） | Eden 区满 | 频繁、速度快（~20-50ms）、STW |
| **Major GC / Full GC** | 整个堆（Young + Old + Metaspace） | 老年代满 / 晋升失败 / 元空间不足 | 罕见、极慢（~秒级）、**全程 STW，性能杀手** |
| **Mixed GC** | 新生代 + 部分老年代 Region | G1GC 特有，并发标记完成后触发 | 平衡延迟与吞吐，G1GC 的设计核心 |

### Minor GC / YGC

- **Minor GC** 和 **YGC（Young Generation GC）** 是**同一个概念**，只是命名角度不同
- Minor GC = 从规模角度（相对于 Major/Full）
- YGC = 从区域角度（回收 Young Gen）
- 在 `jstat -gc` 输出中以 `YGC` / `YGCT` 列标识

### Full GC

- 最重的 STW 事件，性能影响最大
- 不同 GC 回收器的 Full GC 触发条件不同（详见下方对比）

---

## 二、按回收器实现分类

JVM 提供了多种垃圾回收器实现，从早期的串行到现代的低延迟并发收集器：

### 1. Serial GC（串行回收器）

| 项目 | 说明 |
|------|------|
| **启用** | `-XX:+UseSerialGC` |
| **JDK 版本** | JDK 1.3+，JDK 9 前是 Client 模式默认 |
| **特点** | 单线程回收，全程 STW |
| **适用** | 堆 < 100MB、单核 CPU、桌面应用、测试环境 |
| **不适用** | 服务端应用、大堆、低延迟要求 |

> 虽然最古老，但在小堆 + 单核场景下开销最小。

### 2. Parallel GC（并行回收器 / 吞吐量优先）

| 项目 | 说明 |
|------|------|
| **启用** | `-XX:+UseParallelGC` |
| **JDK 版本** | JDK 1.4.3+，**JDK 8 默认** |
| **特点** | 多线程并行回收，**追求高吞吐量** |
| **适用** | 批处理任务、科学计算、后台作业 |
| **不适用** | 低延迟交互式应用（全程 STW） |

> **吞吐量 = 用户代码运行时间 / (用户代码运行时间 + GC 时间)**。Parallel GC 目标是最大化这个值。

### 3. CMS（Concurrent Mark Sweep）— ⚠️ 已移除

| 项目 | 说明 |
|------|------|
| **启用** | `-XX:+UseConcMarkSweepGC` |
| **生命周期** | JDK 1.4.2~JDK 8 主流，**JDK 9 弃用，JDK 14 正式移除** |
| **特点** | 并发标记+清除，减少 STW 时间 |
| **致命缺陷** | 内存碎片化、Concurrent Mode Failure → 退化为 Serial Old（FGC 极慢） |
| **替代** | **G1 GC** |

> **面试高频**：CMS 为什么被移除？① 内存碎片 ② Concurrent Mode Failure ③ 无法处理浮动垃圾 ④ G1 全面超越。

### 4. G1 GC（Garbage-First）— 🏆 当前默认

| 项目 | 说明 |
|------|------|
| **启用** | `-XX:+UseG1GC`（**JDK 9+ 默认**） |
| **JDK 版本** | JDK 7u4 实验 ~ JDK 9 正式默认 ~ 至今 |
| **设计思想** | Region 化堆 + 优先回收垃圾最多的 Region（"Garbage-First"） |
| **Pause 目标** | 可配置 `-XX:MaxGCPauseMillis=200`，通常 50~200ms |
| **最大堆** | 约 32TB |

**G1GC 工作流程**：
```
初始标记(STW) → 并发标记 → 最终标记(STW) → 筛选回收(STW)
   ↓              ↓              ↓              ↓
标记GC Roots  标记存活对象    处理引用更新   优先回收垃圾最多
                                              的 Region（Mixed GC）
```

> **核心技术**：
> - 堆划分为 **Region**（1~32MB），新生代/老年代不再连续
> - **Mixed GC**：一次回收新生代 + 部分老年代 Region
> - **Remembered Set**：记录跨 Region 引用，避免全堆扫描
> - **SATB（Snapshot-At-The-Beginning）**：并发标记时保证正确性

### 5. ZGC（低延迟回收器）

| 项目 | 说明 |
|------|------|
| **启用** | `-XX:+UseZGC` |
| **JDK 版本** | JDK 11 实验 ~ JDK 15 正式 ~ 至今 |
| **Pause 时间** | **< 10ms**（与堆大小无关！） |
| **最大堆** | **16TB+** |
| **设计思想** | 染色指针（Colored Pointers）+ 读屏障（Load Barriers），几乎全并发 |

**核心技术**：
- **染色指针**：利用 64 位指针的高 4 位存储 GC 元数据（Finalizable、Remap、Marked0、Marked1）
- **读屏障**：应用线程读取对象时，如果对象正在被 GC 移动，屏障修正引用
- **全并发**：除初始标记/最终标记外，所有阶段并发，STW 极短

**JDK 21+ 变化**：
- 引入 **Generational ZGC**（实验性 `-XX:+ZGenerational`）
- 分代后吞吐量显著提升，JDK 23 基准测试显示 CPU 开销甚至低于 G1

### 6. Shenandoah GC

| 项目 | 说明 |
|------|------|
| **启用** | `-XX:+UseShenandoahGC` |
| **JDK 版本** | JDK 12 实验 ~ 至今 |
| **Pause 时间** | **< 10ms** |
| **设计思想** | Brooks Pointer + 读写屏障，全并发压缩 |
| **注意** | **Oracle JDK 不包含**，仅 OpenJDK / Red Hat 构建 |

**与 ZGC 的关键区别**：
| 维度 | ZGC | Shenandoah |
|------|-----|------------|
| 屏障 | 读屏障（Load Barrier） | 读屏障 + 写屏障（Load + Store Barrier） |
| 指针技术 | 染色指针（Colored Pointer） | Brooks Pointer（转发指针） |
| 堆压缩 | 全并发 | 全并发 |
| 分代支持 | JDK 21+ Generational ZGC（实验） | JDK 24+ Generational Shenandoah（实验），**JDK 25 正式稳定** |

### 7. Epsilon GC（无操作回收器）

| 项目 | 说明 |
|------|------|
| **启用** | `-XX:+UseEpsilonGC` |
| **JDK 版本** | JDK 11+ |
| **行为** | **分配对象但不回收**，堆满即 OOM |
| **适用** | 性能测试、极限压测、生命周期极短的微服务 |

> 不是"真正的" GC，而是用于测量 GC 开销的基准线工具。

---

## 三、GC 回收器总对比

| 回收器 | 原理 | JDK 版本 | 默认? | Pause 时间 | 吞吐量 | 堆大小 | 适用场景 |
|--------|------|----------|:-----:|:----------:|:-----:|:------:|---------|
| **Serial** | 单线程 STW | 始终 | 否(JDK9+) | ~秒级 | 低(单核) | <100MB | 桌面/测试 |
| **Parallel** | 多线程 STW | 始终 | **JDK8 默认** | ~秒级 | **最高** | ~8GB | 批处理/离线 |
| **CMS** ❌ | 并发标记清除 | 1.4~13 | 否 | ~百毫秒 | 中 | ~8GB | ❌ 已移除 |
| **G1 🔥** | Region + Mixed GC | JDK7+ | **JDK9+ 默认** | ~50-200ms | 中高 | ~32TB | **通用服务端** |
| **ZGC** | 染色指针 + 读屏障 | JDK11+ | 否 | **<10ms** | 中 | **16TB+** | 大堆低延迟 |
| **Shenandoah** | Brooks 指针 + 读写屏障 | JDK12+ | 否 | **<10ms** | 中 | TB级 | OpenJDK 低延迟 |
| **Epsilon** | 不回收 | JDK11+ | 否 | N/A | N/A | 不限 | 基准测试 |

> **CMS** 已在 JDK 14 移除，仅作了解，面试中会被问及"为什么被 G1 替代"。

---

## 四、GC 选型指南（JDK 21+）

| 场景 | 推荐 GC | 理由 |
|------|---------|------|
| **堆 < 4GB，单机应用** | Serial / Parallel | 最小开销 |
| **批处理、离线计算** | **Parallel GC** | 吞吐量优先 |
| **通用 Web 服务 / 微服务** | **G1 GC**（默认） | 平衡延迟与吞吐，自调优 |
| **大堆 > 32GB，低延迟要求** | **ZGC** | <10ms 停顿，TB级堆 |
| **金融、游戏、广告（超低延迟）** | **ZGC** 或 **Shenandoah** | <10ms 目标 |
| **高分配率 + 低延迟** | **Generational ZGC** (JDK 21+) / **Generational Shenandoah** (JDK 25+) | 分代后吞吐显著提升 |
| **基准测试 / 压测** | Epsilon GC | 消除 GC 干扰 |

### 快速记忆口诀

```
小堆串行大堆并    （Serial 小堆，Parallel 大堆）
服务均衡用 G1     （通用服务端默认）
超低延迟 ZGC 行   （<10ms 停顿）
CMS 已死好好停    （JDK 14 已移除）
```

---

## 五、启动参数速查

```bash
# Serial GC
-XX:+UseSerialGC

# Parallel GC（JDK 8 默认）
-XX:+UseParallelGC
-XX:ParallelGCThreads=<N>       # GC 线程数
-XX:+UseAdaptiveSizePolicy      # 自适应大小调整（默认开启）

# G1 GC（JDK 9+ 默认）
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200        # 目标停顿时间（默认 200ms）
-XX:G1HeapRegionSize=<N>        # Region 大小（1~32MB）
-XX:G1NewSizePercent=5          # 新生代初始占比

# ZGC
-XX:+UseZGC
-XX:ZAllocationSpikeTolerance=2.0  # 分配波动容忍度

# Generational ZGC（JDK 21+ 实验）
-XX:+UseZGC -XX:+ZGenerational

# Shenandoah（OpenJDK 专属）
-XX:+UseShenandoahGC

# Epsilon（无操作/基准测试）
-XX:+UseEpsilonGC
```

---

## 六、面试高频题

1. **Minor GC 和 YGC 是什么关系？** → 完全等价，不同命名角度
2. **G1 GC 为什么能替代 CMS？** → CMS 碎片化 + Concurrent Mode Failure，G1 无碎片 + 可预测停顿
3. **ZGC 如何做到 <10ms？** → 染色指针 + 读屏障 + 全并发设计
4. **Parallel GC 和 G1 GC 如何选择？** → 吞吐优先选 Parallel，延迟优先选 G1
5. **CMS 为什么被移除？** → 内存碎片、CMF 退化为串行 FGC、浮动垃圾无法处理
6. **Generational ZGC 和 Shenandoah 的前景？** → 分代是大趋势，JDK 21+ 逐步成熟

---

## 参考链接

- [[JVM-堆内存分代模型]] — 堆内存结构，GC 回收的基础
- [[JVM-对象创建与内存分配]] — 对象创建流程、TLAB、对象头
- [[GC-Roots详解]] — 可达性分析、GC Roots 分类
- [[jstat-命令详解]] — GC 监控与指标解读
- [[JVM-OOM排查指南]] — OOM 类型与排查工具
- [[JVM内存溢出预防]] — OOM 预防策略
- [[快手电商-一面-19题总结]] — 面试实战总结
