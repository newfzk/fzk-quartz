---
title: JVM OOM 排查指南
type: basic-note
date: 2026-06-03
tags: java, jvm, OOM, 排查, GC
---

# JVM OOM 排查指南

## OOM 核心原因

1. **老年代内存泄漏**：大量对象无法被 GC 回收，持续进入老年代直至占满
2. **内存配置不合理**：Survivor 区配置失衡，对象过早晋升老年代
3. **Full GC 失效**：老年代存在不可回收对象，Full GC 无法释放内存

## 排查步骤

### 1. 获取内存快照

```bash
# 查看 Java 进程 ID
jps -l | grep 服务关键词

# 生成堆快照
jmap -dump:format=b,file=dump.hprof [pid]
```

分析工具：Eclipse MAT（推荐）→ 执行 "Leak Suspects" 分析

### 2. 检查 Full GC 日志

确保 JVM 启动参数中包含：

```bash
-Xloggc:/var/log/java/gc.log -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+PrintHeapAtGC
```

重点关注：Full GC 前后老年代使用量是否下降（不下降 = 内存泄漏）

### 3. 分析业务代码

常见泄漏场景：

- **静态集合未清理**：`static HashMap` 长期 put 但未 remove
- **ThreadLocal 未 remove**：线程池 + ThreadLocal 导致对象无法回收
- **连接未关闭**：数据库连接池、Redis 客户端未释放
- **大对象直接进入老年代**：一次性读取大文件、生成大 JSON

### 4. 临时缓解 JVM 配置

```bash
# 调整年轻代与老年代比例
-Xms4G -Xmx4G -Xmn2G

# 优化 Survivor 区比例（默认 8:1，改为 4:1）
-XX:SurvivorRatio=4

# G1 收集器：提前触发老年代回收
-XX:InitiatingHeapOccupancyPercent=70
```

### 5. 长期监控

- 工具：Prometheus + Grafana / Arthas
- 监控指标：老年代使用率 > 85% 报警、Full GC 次数 > 5次/分钟 报警

## 相关笔记

- [[jstat-命令详解]]
- [[JVM-堆内存分代模型]]
- [[JVM内存溢出预防]]