---
title: VictoriaLogs概述
type: basic-note
date: 2025-05-21
tags: victorialogs, log, overview
---

# VictoriaLogs概述

> 开源协议： Apache v2.0

- github: <https://github.com/VictoriaMetrics/victorialogs-datasource>
- 官方文档：<https://docs.victoriametrics.com/victorialogs/>

VictoriaLogs 是一个开源日志存储数据库 来自于 [VictoriaMetrics](https://github.com/VictoriaMetrics/VictoriaMetrics/)

- 可以从众多日志收集器中接受日志，具体见 [VictoriaLogs - Data ingestion](https://docs.victoriametrics.com/victorialogs/data-ingestion/) 其中包括：
  - [Fluentbit](https://docs.victoriametrics.com/victorialogs/data-ingestion/fluentbit/)
  - 等
- 易于使用，配置简单 具体见 [VictoriaLogs - Quick Start](https://docs.victoriametrics.com/victorialogs/quickstart/)
- 简单强大的查询语言 支持跨所有日志字段的全文搜索 具体见 [LogsQL docs](https://docs.victoriametrics.com/victorialogs/logsql/)
- 支持和grafana集成
- 多租户 可能需要结合 `vmauth` 或其他鉴权插件
