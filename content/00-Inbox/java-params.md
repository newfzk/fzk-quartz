---
title: java-params
type: basic-note
date: 2025-12-15
tags: java, params, command
---

# java-params

`java [params] -jar <jar文件路径>`

常用参数：

- `-D`: 如`-Dloader.path`
- 堆内存相关启动参数
  - `-XX:MaxRAMPercentage=75.0`: 最大堆内存占容器（虚拟机）总内存的75%
  - `-XX:InitialRAMPercentage=50.0`: 初始堆内存占比
  - `-XX:MinRAMPercentage=25.0`: 最小堆内存占容器（虚拟机）总内存的25%
