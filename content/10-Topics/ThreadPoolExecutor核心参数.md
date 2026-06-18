---
title: ThreadPoolExecutor 核心参数
date: 2026-06-01
aliases:
  - ThreadPoolExecutor参数
  - 线程池参数
  - 线程池工作流程
  - 线程池队列
related:
  - "[[进程线程协程]]"
  - "[[线程池拒绝策略]]"
  - "[[Java线程池创建方式]]"
  - "[[线程池监控与调优]]"
  - "[[JVM内存溢出预防]]"
tags:
  - language/java
  - topic/java/并发
  - topic/线程池
status: to-review
---

# ThreadPoolExecutor 核心参数

> [!abstract]
> `ThreadPoolExecutor` 是 Java 线程池的核心实现类，理解其七大连参是掌握线程池的**基本功**。

## 构造方法签名

```java
public ThreadPoolExecutor(
    int corePoolSize,                // 核心线程数
    int maximumPoolSize,             // 最大线程数
    long keepAliveTime,              // 空闲线程存活时间
    TimeUnit unit,                   // 时间单位
    BlockingQueue<Runnable> workQueue,  // 工作队列
    ThreadFactory threadFactory,         // 线程工厂
    RejectedExecutionHandler handler     // 拒绝策略（见 [[线程池拒绝策略]]）
)
```

## 七大连参详解

### 1. corePoolSize（核心线程数）
- 即使空闲也保留在线程池中的线程数量
- 除非设置了 `allowCoreThreadTimeOut(true)`，否则核心线程不会因空闲被回收

### 2. maximumPoolSize（最大线程数）
- 线程池允许的最大线程数量
- 当工作队列满且当前线程数 < maximumPoolSize 时，创建新线程

### 3. keepAliveTime + TimeUnit（空闲存活时间）
- 非核心线程空闲超过该时间会被回收
- 核心线程可通过 `allowCoreThreadTimeOut(true)` 启用超时回收

### 4. workQueue（工作队列）
- 存储等待执行任务的阻塞队列
- 见下方[工作队列类型](#工作队列类型)

### 5. threadFactory（线程工厂）
- 用于创建新线程，可自定义线程名称、是否为守护线程等

### 6. handler（拒绝策略）
- 当线程池和队列都满时执行
- 详见 [[线程池拒绝策略]]

## 工作流程

```
提交任务
    ↓
核心线程数是否已满？
├── 否 → 创建新核心线程执行任务
└── 是 → 工作队列是否已满？
          ├── 否 → 将任务加入队列
          └── 是 → 线程数是否达到最大？
                    ├── 否 → 创建非核心线程执行任务
                    └── 是 → 执行拒绝策略（[[线程池拒绝策略]]）
```

> [!tip] 流程理解要点
> 线程池不是在 corePoolSize 满了后立即创建新线程，而是**先将任务放入队列**。只有当**队列也满了**，才会创建非核心线程。这种设计优先考虑"排队等待"而非"创建新线程"。

## 工作队列类型

| 队列类型 | 特点 | 风险 |
|---------|------|------|
| `ArrayBlockingQueue` | 有界队列，固定容量 | 队列满时触发拒绝策略 |
| `LinkedBlockingQueue` | 默认无界（`Integer.MAX_VALUE`） | 可能导致 OOM（见 [[JVM内存溢出预防]]） |
| `SynchronousQueue` | 直接移交，无缓冲 | 需要足够的最大线程数 |
| `PriorityBlockingQueue` | 优先级队列 | 无界，可能 OOM |

> [!warning] 队列选择关乎系统稳定性
> 使用无界队列时，一旦任务生产速度 > 消费速度，队列无限增长 → **内存溢出（OOM）**。因此**生产环境强烈建议使用有界队列**。

## 线程池的优势

| 特性 | 说明 |
|------|------|
| **降低开销** | 避免线程创建/销毁的频繁开销 |
| **提高响应** | 任务到达时无需等待线程创建 |
| **可控并发** | 限制最大并发数，防止资源耗尽 |
| **便于管理** | 统一管理线程生命周期和任务队列 |

> [!seealso] 继续阅读
> - [[线程池拒绝策略]] — 四种策略详解与选型
> - [[Java线程池创建方式]] — 手动创建 vs `Executors` 工厂方法
> - [[线程池监控与调优]] — 监控指标与动态调整
> - [[Java虚拟线程]] — JDK 21+ 虚拟线程对线程池的冲击
