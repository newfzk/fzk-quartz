---
title: 日志导入导出-VictoriaLogs
type: basic-note
date: 2025-05-29
tags:
  - topic/VictoriaLogs
status: to-review
---

# 日志导入导出-VictoriaLogs

## 导出

[日志导出](https://docs.victoriametrics.com/victorialogs/faq/#how-to-export-logs-from-victorialogs)

向`/select/logsql/query`发送包含所需过滤器的查询, 
VictoriaLogs 将以[ndjson 即 JSON lines](https://jsonlines.org/)的形式返回请求的日志(VictoriaLogs will return the requested logs as a stream of JSON lines)

```shell
curl http://192.168.235.41:39428/select/logsql/query -d 'query=kubernetes.container_name:"business-sys" _time:10m | limit 1'
# 结果如下
{"_time":"2025-05-29T07:45:40.647157Z","_stream_id":"00000000000000005c3492406c18af402ea1baa81849b7a9","_stream":"{stream=\"stderr\"}","_msg":"Picked up JAVA_TOOL_OPTIONS: -XX:+StartAttachListener\n","stream":"stderr","kubernetes.annotations.kubectl.kubernetes.io/restartedAt":"2025-04-27T18:13:03+08:00","kubernetes.container_hash":"harbor.registry.com/t1/business-sys@sha256:f104c56c8739c8824cdc7574cee4126a2053b4c7e43838fc8f488fcd4b0df9fe","kubernetes.container_image":"harbor.registry.com/t1/business-sys:163-108610","kubernetes.container_name":"business-sys","kubernetes.docker_id":"bf775d662e8f33aa1834c158b7153c753b52ad0a1dd44963019d95c0852a5345","kubernetes.host":"192.168.235.43","kubernetes.labels.app":"business-sys","kubernetes.labels.pod-template-hash":"6ff99478d6","kubernetes.labels.type":"rs10","kubernetes.namespace_name":"t1","kubernetes.pod_id":"faaee60a-176a-42aa-8a9e-3561921bf320","kubernetes.pod_ip":"172.20.2.42","kubernetes.pod_name":"business-sys-6ff99478d6-nfsgq","time":"2025-05-29T07:45:40.647157626Z"}
```

## 导入

- es批量api
- loki json api
- [json流 api （ndjson）](https://docs.victoriametrics.com/victorialogs/data-ingestion/#loki-json-api)

通过`/insert/jsonline`端点接收ndjson格式的日志数据示例如下：

```shell
echo '{ "log": { "level": "info", "message": "hello world" }, "date": "0", "stream": "stream1" }
{ "log": { "level": "error", "message": "oh no!" }, "date": "0", "stream": "stream1" }
{ "log": { "level": "info", "message": "hello world" }, "date": "0", "stream": "stream2" }
' | curl -X POST -H 'Content-Type: application/stream+json' --data-binary @- \
 'http://localhost:9428/insert/jsonline?_stream_fields=stream&_time_field=date&_msg_field=log.message'
```

## 相关笔记

- [[VictoriaLogs-关键概念]]