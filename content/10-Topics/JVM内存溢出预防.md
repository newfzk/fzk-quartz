---
title: JVM 内存溢出（OOM）预防策略
date: 2026-06-01
aliases:
  - OOM预防
  - 内存溢出预防
  - OutOfMemoryError
  - 堆溢出
  - 栈溢出
related:
  - "[[ThreadPoolExecutor核心参数]]"
  - "[[Java线程池创建方式]]"
  - "[[线程池监控与调优]]"
tags:
  - language/java
  - topic/JVM
  - topic/java/并发
status: to-review
---

# JVM 内存溢出（OOM）预防策略

> [!abstract]
> 线程池使用不当是 OOM 的常见诱因之一。理解 **OOM 的类型**与**预防手段**，是从"会用线程池"到"用好线程池"的关键一步。

## OOM 常见类型

| 类型 | 原因 | 典型错误信息 |
|------|------|-------------|
| **堆溢出** | 对象无法回收，内存耗尽 | `OutOfMemoryError: Java heap space` |
| **栈溢出** | 递归过深或线程过多 | `OutOfMemoryError: unable to create new native thread` |
| **元空间溢出** | 类加载过多 | `OutOfMemoryError: Metaspace` |

### 栈溢出与线程池的关系

> `unable to create new native thread` 是线程池场景中容易忽略的 OOM。

每个平台线程约占用 ~1MB 栈空间（`-Xss` 参数控制）。当线程池的 `maximumPoolSize` 设置过大，或使用 `Executors.newCachedThreadPool()`（最大容量为 `Integer.MAX_VALUE`）时，可能创建大量线程导致**操作系统无法再创建新的线程**。

## 线程池相关的 OOM 预防

```java
// ✅ 安全的线程池配置
public class SafeThreadPoolConfig {
    public static ExecutorService createSafeThreadPool() {
        int corePoolSize = Runtime.getRuntime().availableProcessors() * 2;
        int maxPoolSize = corePoolSize * 2;

        return new ThreadPoolExecutor(
            corePoolSize,
            maxPoolSize,
            60L,
            TimeUnit.SECONDS,
            new LinkedBlockingQueue<>(1000),   // 有界队列（关键！）
            Executors.defaultThreadFactory(),
            new ThreadPoolExecutor.CallerRunsPolicy()  // 降级策略
        );
    }
}
```

### 三条核心防线

```
任务提交 → [第一道] 有界队列 → [第二道] 最大线程限制 → [第三道] 拒绝策略
             防止队列无限增长      防止无限创建线程        降级/抛异常
```

1. **有界队列** — 队列有容量上限，不会无限增长
2. **合理的线程上限** — 避免创建过多线程压垮系统
3. **拒绝策略** — [[线程池拒绝策略]] 是最后的保护手段

## 数据库查询的内存优化

大量数据一次性加载到内存是 OOM 的另一大元凶。

### 方式一：JDBC 流式查询

```java
public class StreamQueryExample {
    // 使用 JDBC 流式查询（逐行读取，不缓存全部结果）
    public void processOrdersStream(Long storeId) throws SQLException {
        String sql = "SELECT id, amount FROM orders WHERE store_id = ?";

        try (Connection conn = getConnection();
             PreparedStatement stmt = conn.prepareStatement(sql,
                 ResultSet.TYPE_FORWARD_ONLY,
                 ResultSet.CONCUR_READ_ONLY)) {

            stmt.setLong(1, storeId);
            stmt.setFetchSize(Integer.MIN_VALUE);  // ⚡ 启用流式获取

            try (ResultSet rs = stmt.executeQuery()) {
                long total = 0;
                while (rs.next()) {
                    total += rs.getLong("amount");
                    // 每行处理，不会将所有结果加载到内存
                }
            }
        }
    }
}
```

> [!tip] `setFetchSize(Integer.MIN_VALUE)` 的含义
> 在 MySQL JDBC 驱动中，设置 fetch size 为 `Integer.MIN_VALUE` 会**逐行读取**结果集，而非一次性将全部结果加载到内存。这是**防止大查询 OOM 的经典手段**。

### 方式二：分页查询

```java
public long calculateSalesWithPagination(Long storeId) {
    long total = 0;
    int pageNum = 0;
    int pageSize = 1000;

    while (true) {
        Page<Order> page = orderRepository.findByStoreId(
            storeId,
            PageRequest.of(pageNum, pageSize)
        );

        if (page.isEmpty()) break;

        total += page.getContent().stream()
            .mapToLong(Order::getAmount)
            .sum();

        pageNum++;
    }
    return total;
}
```

> [!tip] 流式 vs 分页选型
> - **流式查询**：适合全表扫描、数据量极大（百万级以上），内存占用最低
> - **分页查询**：适合需要聚合计算（如 COUNT、SUM）的场景，每页控制内存

## JVM 参数配置建议

```bash
# 生产环境推荐配置
java -Xms4g \                      # 初始堆大小
     -Xmx8g \                      # 最大堆大小（根据服务器配置调整）
     -XX:MetaspaceSize=256m \
     -XX:MaxMetaspaceSize=512m \
     -XX:+UseG1GC \                # 使用 G1 垃圾收集器
     -XX:MaxGCPauseMillis=200 \    # 最大GC停顿时间（毫秒）
     -XX:+HeapDumpOnOutOfMemoryError \   # OOM时自动生成堆转储
     -XX:HeapDumpPath=/tmp/heapdump.hprof \
     -jar application.jar
```

### 参数说明

| 参数 | 作用 |
|------|------|
| `-Xms` / `-Xmx` | 堆初始/最大大小（建议设为相同值，避免动态调整） |
| `-XX:+UseG1GC` | G1 收集器，适合大堆内存，暂停时间可控 |
| `-XX:+HeapDumpOnOutOfMemoryError` | OOM 时自动 dump，事后分析根因 |
| `-Xss` | 线程栈大小（默认 ~1MB，减少可缓解线程数上限） |

## 防 OOM 完整 Checklist

- [ ] 使用有界队列（设置合理容量）
- [ ] 避免使用 `Executors.newCachedThreadPool()`
- [ ] 设置合理的 `maximumPoolSize`，防止创建过多线程
- [ ] 数据库查询使用分页 / 流式读取
- [ ] 及时释放不再使用的对象引用
- [ ] 设置合理的 JVM 堆内存大小
- [ ] 配置 OOM 自动堆转储
- [ ] 监控队列积压（见 [[线程池监控与调优]]）

> [!seealso] 相关笔记
> - [[ThreadPoolExecutor核心参数]] — 工作队列类型与风险
> - [[Java线程池创建方式]] — 线程池创建的最佳实践
> - [[线程池监控与调优]] — 监控队列积压是预防 OOM 的关键
> - [[线程池拒绝策略]] — 拒绝策略是 OOM 的最后一道防线
