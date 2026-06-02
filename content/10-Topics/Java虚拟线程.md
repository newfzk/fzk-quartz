---
title: Java 虚拟线程（Virtual Threads）
date: 2026-06-01
tags:
  - topic/并发控制
  - topic/并发编程基础
  - language/java
  - topic/线程池
aliases:
  - 虚拟线程
  - Virtual Threads
  - Project Loom
  - 协程Java实现
  - 纤程
related:
  - "[[进程线程协程]]"
  - "[[ThreadPoolExecutor核心参数]]"
  - "[[Java线程池创建方式]]"
---

# Java 虚拟线程（Virtual Threads）

> [!quote] JDK 21（2023年9月 LTS）正式发布虚拟线程
> 虚拟线程（Virtual Threads）是 [[进程线程协程#协程（Coroutine）|协程]] 在 Java 中的实现，由 Project Loom 孵化而来。

## 什么是虚拟线程？

虚拟线程是 **JVM 管理的轻量级线程**，由 JVM 在少量平台线程（载体线程/Carrier Thread）上调度，而非 OS 内核。

- **平台线程**（Platform Thread）= 传统 `java.lang.Thread`，映射为 OS 线程
- **虚拟线程**（Virtual Thread）= 由 JVM 调度的轻量级线程，多个虚拟线程复用一个平台线程

## 如何使用

### 方式一：虚拟线程执行器（推荐）

```java
// JDK 21+：每个任务一个虚拟线程，无需池化
ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();

// 使用方式与普通线程池完全一致
executor.submit(() -> {
    // 处理 I/O 密集型任务
    String result = httpClient.send(request, BodyHandlers.ofString());
    return result;
});
```

### 方式二：直接启动

```java
Thread vThread = Thread.startVirtualThread(() -> {
    System.out.println("Hello from virtual thread!");
});
```

### 方式三：Thread.Builder

```java
Thread vThread = Thread.ofVirtual()
    .name("my-virtual-thread")
    .start(() -> {
        // 任务逻辑
    });
```

## 虚拟线程 vs 平台线程

| 特性 | 平台线程（OS 线程） | 虚拟线程 |
|:---|:---|:---|
| **调度** | OS 内核调度 | JVM 用户态调度 |
| **栈大小** | ~1MB（固定） | ~几 KB（可动态扩展） |
| **最大数量** | 几千 ~ 几万 | **数百万** |
| **创建成本** | 高（系统调用） | 极低（纯内存操作） |
| **阻塞代价** | 阻塞 = 线程被挂起（浪费 OS 资源） | 阻塞 = 切换到其他虚拟线程（零成本） |
| **适用场景** | CPU 密集型 | **大量 I/O 等待型任务** |

## 虚拟线程的本质：面向 I/O 密集型场景

虚拟线程的核心思想是：**每个请求/任务一个线程，而不是从池中借用一个线程**。

```java
// 传统方式：线程池限制并发
ExecutorService pool = Executors.newFixedThreadPool(200);
for (Request request : requests) {
    pool.submit(() -> handleRequest(request));
}

// 虚拟线程方式：每个请求一个虚拟线程，无需池化
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (Request request : requests) {
        executor.submit(() -> handleRequest(request));
    }
}
```

## 局限性

> [!warning] 虚拟线程 ≠ 万能药

| 限制 | 说明 |
|------|------|
| **CPU 密集型** | 不适合，无法利用更多 CPU 核心，仍需平台线程池控制并发 |
| **synchronized pinning** | 虚拟线程在 `synchronized` 块内无法被卸载，会钉住载体线程（JDK 21 已部分修复：`ReentrantLock` 无此问题） |
| **ThreadLocal** | 虚拟线程支持 `ThreadLocal`，但创建海量虚拟线程时可能导致内存压力（慎用） |
| **native 方法** | 调用 JNI 时虚拟线程会被钉住 |

## 最佳实践

```java
// ✅ 虚拟线程 + 信号量控制并发数（混合模式）
// 适用于既有 I/O 等待又是 CPU 密集的混合场景
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    Semaphore semaphore = new Semaphore(100);  // 控制并发度
    for (Task task : tasks) {
        executor.submit(() -> {
            semaphore.acquire();
            try {
                task.process();
            } finally {
                semaphore.release();
            }
        });
    }
}
```

> [!seealso] 相关笔记
> - [[进程线程协程]] — 进程、线程、协程的基础概念
> - [[Java线程池创建方式]] — 传统平台线程池的创建方式
> - [[ThreadPoolExecutor核心参数]] — 线程池构造参数详解
