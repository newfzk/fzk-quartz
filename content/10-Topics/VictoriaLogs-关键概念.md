---
title: VictoriaLogs 关键概念
type: basic-note
date: 2025-05-21
tags:
  - topic/VictoriaLogs
  - topic/日志
status: to-review
---

# VictoriaLogs 关键概念

> 官方文档：<https://docs.victoriametrics.com/victorialogs/keyconcepts/>

## 数据模型 Data Model

VictoriaLogs 支持结构化和非结构化日志。
每个日志条目`entry`必须包含至少一个日志消息字段`_msg`。
可以向日志条目添加任意数量的附加`key=value`字段。
单个日志条目可以表示为由字符串键、字符串值组成的单级JSON对象。如：

```json
{
  "job": "my-app",
  "instance": "host123:4567",
  "level": "error",
  "client_ip": "1.2.3.4",
  "trace_id": "1234-56789-abcdef",
  "_msg": "failed to serve the client request"
}
```

空值和不存在的值处理相同，下例中的几个json对象等价，都是只有一个非空属性`_msg`

```json
{
  "_msg": "foo bar",
  "some_field": "",
  "another_field": ""
},

{
  "_msg": "foo bar",
  "third_field": "",
},

{
  "_msg": "foo bar",
}
```

VictoriaLogs 在进行 数据收集（`data ingestion`） 时会自动将多级（嵌套）JSON转换为单级JSON，规则如下：

- 嵌套的字典使用`.`连接key实现扁平化 示例如下

```json
{
  "host": {
    "name": "foobar",
    "os": {
      "version": "1.2.3"
    }
  }
},

{
  "host.name": "foobar",
  "host.os.version": "1.2.3"
}
```

- 数组、数字、布尔值都被转换为字符串 示例如下

```json
{
  "tags": ["foo", "bar"],
  "offset": 12345,
  "is_error": false
},

{
  "tags": "[\"foo\", \"bar\"]",
  "offset": "12345",
  "is_error": "false"
}
```

字段名和字段值中可以包含任意字符。
这些字符必须在数据收集时根据[JSON字符串编码](https://www.rfc-editor.org/rfc/rfc7159.html#section-7)进行编码。
Unicode字符必须使用`UTF-8`编码进行编码。

```json
{
  "field with whitespace": "value\nwith\nnewlines",
  "Поле": "价值"
}
```

VictoriaLogs 会自动索引所有收集日志中的所有字段。
这使得可以在所有字段上进行全文搜索（通过`LogSQL`查询）。

VictoriaLogs 除了任意其他字段外，还支持以下特殊字段：

- [VictoriaLogs 关键概念](#victorialogs-关键概念)
  - [数据模型 Data Model](#数据模型-data-model)
    - [消息字段 \_msg](#消息字段-_msg)
    - [时间字段 \_time](#时间字段-_time)
    - [流字段 \_stream](#流字段-_stream)
    - [其他自定义字段](#其他自定义字段)

### 消息字段 _msg

每个收集的日志条目都**必须**包含`_msg`字段，其中包含实际的日志消息。
换言之， VictoriaLogs 的最小日志条目如下：

```json
{
  "_msg": "some log message"
}
```

可以通过HTTP查询请求，另外指定其他字段起到`_msg`的作用 详见：<https://docs.victoriametrics.com/victorialogs/data-ingestion/#http-parameters>

### 时间字段 _time

收集日志条目的时间戳。该字段必须符合以下格式之一：

- [ISO8601](https://en.wikipedia.org/wiki/ISO_8601)或[RFC3339](https://www.rfc-editor.org/rfc/rfc3339)
  - 如 `2023-06-20T15:32:10Z`
  - 或 `2023-06-20 15:32:10.123456789+02:00`
  - 如果缺少时区信息（如 `2023-06-20 15:32:10`） 则时间将根据运行 VictoriaLogs 的主机本地时区进行解析
- Unix 时间戳 秒、毫秒、微秒或纳秒级 如`1686026893` (seconds), `1686026893735` (milliseconds 毫秒), `1686026893735321` (microseconds 微秒) or `1686026893735321098` (nanoseconds 纳秒)

可以通过HTTP查询请求，另外指定其他字段起到`_time`的作用 详见：<https://docs.victoriametrics.com/victorialogs/data-ingestion/#http-parameters>

> 如果 `_time` 字段缺失或等于`0` 则自动使用数据收集时间作为日志条目的时间戳

`_time`字段将被用于[时间过滤](https://docs.victoriametrics.com/victorialogs/logsql/#time-filter)，以便快速将搜索范围缩小到选定的时间区间。

### 流字段 _stream

一些结构化日志字段可以唯一标识生成日志的应用示例。
这可能是单个字段，如`instance="host123:456"`，也可能是多个字段，
如:`{kubernetes.namespace="cloud-new", kubernetes.node.name="worker01" ,kubernetes.pod.name="business-dev-xxxxx-xxxxx", kubernetes.container.name="business-dev"}`

来自单个应用示例的日志条目形成 VictoriaLogs 中的日志流`log stream`。
VictoriaLogs 优化了单个日志流的存储和查询。好处如下：

- 减少磁盘使用
- 查询性能提升

每个收集的日志条目都与一个日志流相关联。每个日志流包含以下两个特殊字段：

- `_stream_id` 日志流的唯一标识符 可以通过[`_stream_id:...`过滤器](https://docs.victoriametrics.com/victorialogs/logsql/#_stream_id-filter)选择该流中的所有日志。
- `_stream` 该字段包含格式类似于 普罗米修斯指标标签 的流标签 格式如：`{host="host-123", app="my-app"}`

例如，如果`host`和`app`字段都与流相关，那么在相应的日志条目的`_stream`字段中也将具有`{host="host-123", app="my-app"}`值。
可以使用流过滤器搜索`_stream`字段

VictoriaLogs 无法自动确定哪些字段可以唯一标识每条日志流 所以默认情况下， `_stream`字段的值为`{}`。
这可能导致资源使用和查询性能不佳。
因此，在数据收集时，建议通过`_stream_fields`查询参数指定流级字段。
如k8s容器中的日志，推荐具有如下字段：

```json
{
  "kubernetes.namespace": "some-namespace",
  "kubernetes.node.name": "some-node",
  "kubernetes.pod.name": "some-pod",
  "kubernetes.container.name": "some-container",
  "_msg": "some log message"
}
```

相应的，需要在数据收集时指定`_stream_fields=kubernetes.namespace,kubernetes.node.name,kubernetes.pod.name,kubernetes.container.name`查询参数，以便正确的将每个容器的日志存储到不同的流中。

**请勿将易发生变化的字段添加到流中**，如`ip`，`user_id`，`trace_id`等。
因为这些可能会导致[高基数问题 high cardinality issues](https://docs.victoriametrics.com/victorialogs/keyconcepts/#high-cardinality)

### 其他自定义字段

每个收集的日志除了`_msg`和`_time`外，还可能包含任意数量的字段，如`trace_id`等。
这些字段可以用于简化和优化搜索查询。如在专用的`trace_id`中进行搜索比在长日志消息中搜索`trace_id`更快。

## 相关笔记

- [[VictoriaLogs-概述]]
- [[LogsQL-查询语言]]