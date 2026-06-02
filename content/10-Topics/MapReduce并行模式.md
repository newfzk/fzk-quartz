---
title: MapReduce 并行模式
date: 2026-06-01
tags:
  - topic/并发控制
  - topic/设计模式
  - topic/线程池
  - language/java
aliases:
  - MapReduce
  - 分治法
  - 分治并行
  - 映射归约
related:
  - "[[ThreadPoolExecutor核心参数]]"
  - "[[Java线程池创建方式]]"
  - "[[JVM内存溢出预防]]"
---

# MapReduce 并行模式

> [!abstract]
> **分治法（Divide and Conquer）**是处理大规模数据的经典并行模式。将大任务分解为多个子任务并行执行（Map），然后合并子结果（Reduce）。

## 模式概述

```
                    Map 阶段
  将大任务分解为多个子任务，并行执行
  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐
  │Task1 │  │Task2 │  │Task3 │  │Task4 │  ...
  └──┬───┘  └──┬───┘  └──┬───┘  └──┬───┘
     ↓         ↓         ↓         ↓
  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐
  │Res1  │  │Res2  │  │Res3  │  │Res4  │  ...
  └──┬───┘  └──┬───┘  └──┬───┘  └──┬───┘
     │         │         │         │
     └─────────┴────┬────┴─────────┘
                    ↓
               Reduce 阶段
          将子结果合并为最终结果
                    ↓
              ┌──────────┐
              │ Final    │
              │ Result   │
              └──────────┘
```

## 适用场景

| 场景 | 说明 | Map | Reduce |
|------|------|-----|--------|
| 批量数据汇总 | 计算数千家门店销售额 | 各门店独立计算 | 汇总所有门店 |
| 日志分析 | 按天分析日志 | 单台机器解析 | 合并统计 |
| 全文索引 | 文档分词建索引 | 单文档处理 | 合并索引 |
| 大数据统计 | 用户行为统计 | 分片处理 | 聚合结果 |

## 实现示例：门店销售计算

```java
import java.util.List;
import java.util.concurrent.*;
import java.util.stream.Collectors;

public class MapReduceSalesCalculator {
    private final ExecutorService executor;

    public MapReduceSalesCalculator() {
        this.executor = new ThreadPoolExecutor(
            Runtime.getRuntime().availableProcessors() * 2,
            Runtime.getRuntime().availableProcessors() * 4,
            60L, TimeUnit.SECONDS,
            new LinkedBlockingQueue<>(500),
            new ThreadPoolExecutor.CallerRunsPolicy()
        );
    }

    // Map 阶段：并行计算各门店销售额
    private List<CompletableFuture<Long>> map(List<Long> storeIds) {
        return storeIds.stream()
            .map(storeId -> CompletableFuture.supplyAsync(
                () -> calculateSingleStore(storeId), executor))
            .collect(Collectors.toList());
    }

    // Reduce 阶段：汇总所有门店销售额
    private long reduce(List<CompletableFuture<Long>> futures) {
        return futures.stream()
            .mapToLong(f -> {
                try {
                    return f.get(5, TimeUnit.MINUTES);
                } catch (InterruptedException | ExecutionException | TimeoutException e) {
                    log.error("子任务执行失败", e);
                    return 0L;  // 容错：失败的门店记为 0
                }
            })
            .sum();
    }

    // 完整的 Map-Reduce 流程
    public long calculateTotalSales(List<Long> storeIds) {
        List<CompletableFuture<Long>> futures = map(storeIds);
        return reduce(futures);
    }

    private long calculateSingleStore(Long storeId) {
        long total = 0;
        int page = 0;
        while (true) {
            List<Order> orders = orderRepository.findByStoreId(storeId, page++, 500);
            if (orders.isEmpty()) break;
            total += orders.stream().mapToLong(Order::getAmount).sum();
        }
        return total;
    }
}
```

> [!tip] Reduce 阶段的容错
> 单个子任务的失败不应导致整体失败。通过 `try-catch` 在 Reduce 阶段捕获异常，失败的任务可以：
> - 记录日志，返回默认值（如上例返回 0）
> - 将失败项加入重试队列
> - 异步重新提交失败的子任务

## 与普通线程池的区别

| 维度 | 普通线程池任务 | MapReduce 模式 |
|:---|:---|:---|
| **任务关系** | 独立任务，互不影响 | Map 阶段任务独立，Reduce 阶段需聚合 |
| **结果收集** | 各自处理各自的结果 | 需要**等待所有子任务完成**后合并 |
| **失败处理** | 单个任务失败不影响其他 | 支持部分失败容错 |
| **超时控制** | 单个任务超时 | 整体 + 子任务双重超时 |
| **代码结构** | 提交即忘 | Map → Wait → Reduce 三个阶段清晰 |

## 结合虚拟线程

在 [[Java虚拟线程|JDK 21 虚拟线程]] 时代，MapReduce 的实现可以更简洁：

```java
// 虚拟线程版 MapReduce — 无需线程池
public long calculateTotalSalesVThread(List<Long> storeIds) {
    try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
        List<Future<Long>> futures = storeIds.stream()
            .map(id -> executor.submit(() -> calculateSingleStore(id)))
            .toList();

        return futures.stream()
            .mapToLong(f -> {
                try { return f.get(); }
                catch (Exception e) { return 0L; }
            })
            .sum();
    }
}
```

> [!seealso] 相关笔记
> - [[ThreadPoolExecutor核心参数]] — 线程池参数与工作流程
> - [[Java线程池创建方式]] — 线程池的创建和实战案例
> - [[Java虚拟线程]] — 虚拟线程版 MapReduce
> - [[JVM内存溢出预防]] — 流式查询避免 OOM
