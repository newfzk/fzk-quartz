---
title: 日志保留策略-VictoriaLogs
type: basic-note
date: 2025-05-29
tags:
  - topic/VictoriaLogs
  - topic/日志
status: to-review
---

# 日志保留策略-VictoriaLogs

## 日志保留时长 retentionPeriod

官方文档：<https://docs.victoriametrics.com/victorialogs/#retention>

默认情况下 将保留最近7天的日志条目 超出将被丢弃

- 收集的日志存储在按天划分的分区目录中 会自动删除超出配置保留期的分区目录
- `-retentionPeriod=8w` 保留8周的日志 可以很大，如100y（年）

## 最大存储空间 假 maxDiskSpaceUsageBytes

官方文档：<https://docs.victoriametrics.com/victorialogs/#retention-by-disk-space-usage>

最大存储空间配置

> 最大存储空间配置的是假上限 victorialogs **至少会保留最近两天的数据**
> 哪怕最近两天的数据总量超出日志上限

- 日志存储总量上限配置：`-retention.maxDiskSpaceUsageBytes=100GiB`
- 超出存储上限后 旧的日分区（目录）将被自动删除
- victorialogs会对日志存储压缩10被以上 这意味着如果日志存储上限为`100GiB` 它可以存储超`1T`字节的未压缩日志

## 相关笔记

- [[VictoriaLogs-概述]]