---
title: JVM 对象创建与内存分配
date: 2026-06-15
aliases:
  - 对象创建流程
  - TLAB
  - 大对象分配
  - PretenureSizeThreshold
related:
  - "[[JVM-堆内存分代模型]]"
  - "[[JVM-OOM排查指南]]"
tags:
  - language/java
  - topic/JVM
  - topic/面试
status: reviewed
---

# JVM 对象创建与内存分配

> JVM 对象的创建过程涉及类加载、内存分配、零值初始化、对象头设置等多个步骤，内存分配通过 TLAB 优化并发竞争。

## 对象创建流程

```
类加载检查 → 分配内存 → 初始化零值 → 设置对象头 → 执行 `<init>` 方法
```

### 步骤详解

1. **类加载检查**：JVM 检查该类是否已被加载、解析、初始化。未加载则先执行类加载过程
2. **分配内存**：从 Java 堆中划分一块确定大小的内存
3. **初始化零值**：将分配到的内存空间初始化为零值（保证字段不赋值也能访问）
4. **设置对象头**：设置对象的 Mark Word、类型指针（Klass Pointer）、是否启用偏向锁等信息
5. **执行 `<init>` 方法**：执行构造方法，按程序员意图初始化

## TLAB（Thread Local Allocation Buffer）

### 作用

- **避免线程竞争**：每个线程在 Eden 区预分配一块私有缓冲区，分配对象时无需加锁
- **提升分配效率**：大部分对象在 TLAB 内分配，只有 TLAB 用完时才需要同步加锁

### 工作原理

1. 线程启动时，在 Eden 区分配一块线程私有的 TLAB 空间
2. 线程内的对象分配优先在 TLAB 中进行（指针碰撞，无需锁）
3. TLAB 空间不足：
   - 小对象：TLAB 剩余空间 < 阈值 → 在当前 TLAB 内分配（浪费剩余空间）
   - 大对象：TLAB 装不下 → 直接在 Eden 加锁分配

### 相关参数

| 参数 | 说明 |
|------|------|
| `-XX:+UseTLAB` | 启用 TLAB（JDK 8 默认开启） |
| `-XX:TLABSize` | 初始 TLAB 大小 |
| `-XX:TLABRefillWasteFraction` | TLAB 再填充时的浪费阈值 |

## 大对象分配策略

### 直接进入老年代

- **`-XX:PretenureSizeThreshold`**：大于该值的对象直接在老年代分配（默认 0，不启用）
- **目的**：避免大对象在 Eden 和 Survivor 之间频繁复制（Stop-The-World）
- **适用场景**：大数组、大字符串、长生命周期的大对象

### 对象年龄判定

- 每经历一次 Minor GC 存活，对象年龄加 1
- 晋升阈值：`-XX:MaxTenuringThreshold`，默认 15（CMS 默认 6）
- **动态年龄判定**：HotSpot 不必须等达到阈值才晋升，Survivor 中同龄对象总大小 > Survivor 的一半时，大于该年龄的对象直接晋升

## 对象头结构

```
Mark Word（标记字段）: 32bit/64bit — 存储哈希码、GC 分代年龄、锁状态标志
Klass Pointer（类型指针）: 32bit/64bit — 指向方法区的类元数据（-XX:+UseCompressedClassPointers 可压缩）
实例数据: 对象的实例字段
对齐填充: 保证对象起始地址是 8 字节的整数倍
```

## 参考链接

- [[快手电商-一面-19题总结]] — Q2 对象从 new 到 GC 全过程
- [[JVM-堆内存分代模型]] — 对象生命周期与分代回收
- [[JVM-OOM排查指南]] — 内存泄漏排查
- [[JVM-GC类型对比]] — 常见 GC 类型与回收器对比
