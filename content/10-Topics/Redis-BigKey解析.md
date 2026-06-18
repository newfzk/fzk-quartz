---
title: Redis BigKey 解析
date: 2026-06-15
aliases:
  - BigKey
  - 大Key
  - 热Key
  - Redis阻塞
  - UNLINK
tags:
  - topic/Redis
  - topic/缓存
  - topic/性能优化
status: to-review
---

# Redis BigKey 解析

> BigKey 是 Redis 中单个 key 的值过大导致的性能问题。Redis 单线程模型下，BigKey 会导致操作阻塞、网络延迟、内存倾斜等一系列问题。

## BigKey 判定标准

| 类型 | 判定标准 | 说明 |
|------|---------|------|
| **String** | Value > 10KB | 大字符串 |
| **Hash** | 元素个数 > 5000 / Len > 10000 | 大哈希 |
| **List** | 元素个数 > 5000 / Len > 10000 | 大列表 |
| **Set** | 元素个数 > 5000 / Len > 10000 | 大集合 |
| **SortedSet** | 元素个数 > 5000 / Len > 10000 | 大有序集合 |

## BigKey 导致的问题

### 1. 操作阻塞（最严重）

- Redis 单线程处理命令，BigKey 的 `GET` / `HGETALL` / `SMEMBERS` 等操作耗时可能 **几百毫秒到几秒**
- 阻塞期间，其他所有请求排队等待 → **吞吐量骤降，延迟飙升**

### 2. 删除阻塞

```bash
# 对一个包含 100 万个元素的 Set 执行 DEL
DEL mybigset  # → Redis 遍历所有元素释放内存，耗时秒级
```

- `DEL`、`SREM`、`HDEL` 等命令的时间复杂度 = O(N)
- 对集合类型操作时，需要遍历每个元素释放内存
- **主线程被阻塞，无法处理其他请求**

### 3. 网络延迟

- 传输 BigKey 占用大量带宽
- 大响应体增加序列化/反序列化开销
- 客户端超时或 OOM 风险

### 4. 内存倾斜

- Redis 集群模式下，BigKey 导致某个分片内存使用量远高于其他分片
- 造成集群负载不均衡

### 5. 数据迁移慢

- 集群扩缩容、主从同步时，BigKey 导致数据重传耗时增加
- 同步延迟影响高可用

## BigKey 发现方法

| 方法 | 命令/工具 | 说明 |
|------|----------|------|
| **redis-cli --bigkeys** | `redis-cli --bigkeys` | 扫描所有 key 并统计分布 |
| **MEMORY USAGE** | `MEMORY USAGE key` | 查看单个 key 的内存使用 |
| **DEBUG OBJECT** | `DEBUG OBJECT key` | 查看 key 的编码和序列化长度 |
| **RDB 分析** | `rdb-tools` / `redis-rdb-tools` | 离线分析 RDB 文件 |
| **MONITOR** | `MONITOR` | 实时监控命令（线上慎用） |

## BigKey 处理方案

### 删除优化

```bash
# 方案一：异步删除（推荐）
UNLINK mybigkey  # → 后台线程回收内存，不阻塞主线程

# 方案二：分批删除
# 使用 SSCAN + SREM 分批处理
SSCAN mybigset 0 COUNT 100  # 每次取 100 个
SREM mybigset "member1" "member2" ...
```

### 拆分策略

| 类型 | 拆分方法 | 示例 |
|------|---------|------|
| **String** | 压缩/分段存储 | 大 JSON 拆成多个小 key |
| **Hash** | 按哈希字段拆分 | `hash:field1`, `hash:field2`... |
| **List/Set** | 按时间/数量切分 | `list:202401`, `list:202402`... |

### 预防措施

1. **合理设计 key 结构**：集合类型设置最大元素数量限制
2. **定期扫描监控**：接入 Redis 慢查询监控和 BigKey 扫描
3. **业务层拆分**：大对象在上层拆解后再写入
4. **设置代理层**：如 Codis/Redis Cluster 代理层可拦截 BigKey 操作

## 参考链接

- [[快手电商-一面-19题总结]] — Q10 Redis BigKey
- [[分布式锁实现]] — Redis 分布式锁
