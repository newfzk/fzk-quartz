---
title: Runtime.getRuntime().availableProcessors()
date: 2026-06-01
tags:
  - topic/JVM
  - topic/并发控制
  - language/java
aliases:
  - availableProcessors
  - Runtime.availableProcessors
  - JVM可用CPU数
  - ActiveProcessorCount
related:
  - "[[Java线程池创建方式]]"
  - "[[ThreadPoolExecutor核心参数]]"
  - "[[JVM内存溢出预防]]"
---

# `Runtime.getRuntime().availableProcessors()`

> [!info] 一句话
> 返回 **JVM 当前可见的 CPU 核心数**，常用于动态计算线程池大小（如 `corePoolSize = CPU核心数 × 2`）。

经测试，当前jdk版本确实返回的是limit.cpu，适配k8s
## 基本用法

```java
int processors = Runtime.getRuntime().availableProcessors();
```

这是 Java 中最常见的 "自动适配机器配置" 的方式，广泛用于：

- 线程池大小计算（`new ThreadPoolExecutor(n processors, ...)`）
- ForkJoinPool 的并行度（`commonPool` 默认据此初始化）
- 并行流 `parallelStream()` 的底层线程数
- `CompletableFuture` 默认线程池大小

## `Runtime` 类

`Runtime` 是 Java 中代表 **JVM 运行时环境** 的单例类。

```java
Runtime runtime = Runtime.getRuntime();
runtime.availableProcessors();  // CPU 核心数
runtime.freeMemory();           // JVM 空闲内存
runtime.totalMemory();          // JVM 总内存
runtime.maxMemory();            // JVM 最大内存（-Xmx）
runtime.gc();                   // 建议 GC
```

> `Runtime.getRuntime()` 是**单例模式**在 JDK 中的经典应用 —— 每个 JVM 进程只有一个 `Runtime` 实例。

## 底层实现

### 普通实现（非容器感知）

内部调用 **JVM 原生方法**，最终读取操作系统信息：

- **Linux**：读取 `/proc/cpuinfo` 或调用 `sysconf(_SC_NPROCESSORS_ONLN)`
- **Windows**：调用 `GetSystemInfo()` API
- **macOS**：调用 `sysctl()` 获取 `hw.ncpu`

```java
// HotSpot JVM 源码示意（简化）
public int availableProcessors() {
    return os::active_processor_count();
}
```

### 容器感知实现（JDK 8u191+ / JDK 10+）

当 `-XX:+UseContainerSupport` 启用时（默认开启），JVM 会额外读取 cgroup 信息：

```java
// JDK 8u191+ 增加容器感知逻辑
// 1. 先读取 cgroup CPU 限额
// 2. 若容器有 limits.cpu 限制，返回限额值
// 3. 若无限制，返回宿主机的 CPU 总数
```

## 返回值的各种情况

```yaml
环境                      availableProcessors()
─────────────────────────────────────────────────
物理机 / 虚拟机 (JDK任意版本) → 真实的物理 CPU 核心数
容器 JDK 8u191 之前        → 宿主机 CPU 总核数 ❌
容器 JDK 8u191+ / JDK 10+  → 容器 limits.cpu 值 ✅
容器 + -XX:ActiveProcessorCount=N → N（完全覆盖） ✅
```

## 在 K8s 中的注意事项

> [!danger] 经典陷阱
> 在容器中直接使用 `Runtime.getRuntime().availableProcessors()` 计算线程池大小，可能得到**宿主机总核数**而非容器的 `limits.cpu`。

```yaml
真实案例：
K8s Pod:    limits.cpu: 2, requests.cpu: 1
宿主机:     32 核
JDK 8u181:  availableProcessors() → 32 ← 危险！
JDK 8u191+: availableProcessors() → 2  ← 正确
```

### 正确做法

```bash
# 方式一：启动参数显式指定（最稳妥，适用于所有 JDK 版本）
java -XX:ActiveProcessorCount=2 -jar app.jar
```

```java
// 方式二：K8s 注入环境变量，代码读取
int processors = Integer.parseInt(
    System.getenv().getOrDefault("CPU_LIMIT",
        String.valueOf(Runtime.getRuntime().availableProcessors()))
);
```

```yaml
# 方式三：K8s Deployment 配置
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
      - env:
        - name: CPU_LIMIT
          valueFrom:
            resourceFieldRef:
              containerName: app
              resource: limits.cpu
```

### JDK 版本验证

```bash
# 查看当前 JVM 是否启用了容器支持
java -XX:+PrintFlagsFinal -version 2>&1 | grep -i container

# 输出示例：
# bool UseContainerSupport = true   ← 容器感知已开启
# int ActiveProcessorCount = 0      ← 0 表示自动检测，N 表示手动指定
```

### 常见误区

| 误区 | 真相 |
|------|------|
| `-Xmx` 能自动适配容器内存 | ✅ 是的（`-XX:+UseContainerSupport` 同时适配内存） |
| `availableProcessors()` 也自动适配 | ✅ JDK 8u191+ 可以，旧版本不行 |
| 设置了 `requests.cpu` 就影响返回值 | ❌ JVM 读取的是 **limits.cpu**（CFS quota），不是 requests |
| `limits.cpu: 1.5` 会返回 1.5 | ❌ 返回 **整型 1**（向下取整），剩余 0.5 核通过 CFS 配额实现 |

> [!seealso] 相关笔记
> - [[Java线程池创建方式]] — `availableProcessors()` 在线程池创建中的实际应用
> - [[ThreadPoolExecutor核心参数]] — 线程池 corePoolSize 与最大线程数如何配置
> - [[JVM内存溢出预防]] — `-Xmx` 同样有容器适配问题
