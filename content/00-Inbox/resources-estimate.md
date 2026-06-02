---
title: VictoriaLogs 工作负载资源估算
type: basic-note
date: 2025-05-30
tags: victorialogs, resources, estimate
---

# VictoriaLogs 工作负载资源估算

## 资源估算

> 按照 1个业务环境 约50个 技术/业务子系统 估计

参考官方文档：[VictoriaLogs 如何口算工作负载所需计算资源](https://docs.victoriametrics.com/victorialogs/faq/#how-to-estimate-the-needed-compute-resources-for-the-given-workload)  

- 磁盘存储空间：`50GB`位于`NFS`服务器
  - 取决于日志数据的数据压缩性和数据保留需求
  - 按 每个子系统 每天 产生 `1GB` 日志估算 每天将生成日志 `50GB`
  - 保留 `7` 天日志 `350GB` 日志
  - 为后续扩展预留 `150GB` 共计 `500GB`
  - VictoriaLogs 日志压缩能力可达 `10倍` 以上，因此，预计需要预留 `50GB` 的磁盘存储空间
  - VictoriaLogs 镜像大小为 `29MB` 忽略不计
- 内存、CPU： 拍脑袋 `request 1核1G limit 4核4g`
  - 取决于 摄入日志的 查询类型 和 查询速率
  - 最近输入日志 轻量级查询 每秒`1000次` 如果使用的日志流过滤器很精确 所消耗的计算资源也很低；
  - 对长时间范围的 重量级查询 没有进行日志过滤器优化或复杂管道处理（类似`select * from table`） 可能需要数百个CPU核和数TB的内存才能快速执行 或 少量CPU核和少量GB内存较长等待时间
  - 集群性能估算时，暂定两个worker各增加`2核2g`的资源要求
  - **推荐估算方法**：启动一个 VictoriaLogs , 令其收集、存储生产环境日志体量的`1%-10%` 在此基础上进行典型查询 记录资源消耗 由此估算全量生产环境下的工作负载；
