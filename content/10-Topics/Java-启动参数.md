---
title: Java 启动参数
type: basic-note
date: 2025-12-15
tags:
  - language/java
  - topic/JVM
status: to-review
---

# Java 启动参数

## 基本语法

```shell
java [params] -jar <jar文件路径>
```

## 常用参数

### 系统属性

- `-D`：设置系统属性，如 `-Dloader.path=lib`

### 堆内存参数

| 参数 | 含义 |
|------|------|
| `-XX:MaxRAMPercentage=75.0` | 最大堆内存占容器总内存的 75% |
| `-XX:InitialRAMPercentage=50.0` | 初始堆内存占比 |
| `-XX:MinRAMPercentage=25.0` | 最小堆内存占比 |

> 使用 `RAMPercentage` 系列参数比 `-Xmx` / `-Xms` 更适合容器化部署，能根据容器内存动态调整。

## 相关笔记

- [[JVM-堆内存分代模型]]
- [[jstat-命令详解]]
- [[JVM-OOM排查指南]]