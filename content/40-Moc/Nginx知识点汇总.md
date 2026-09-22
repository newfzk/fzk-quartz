---
title: Nginx 知识点汇总（MOC）
date: 2026-07-10
tags:
  - topic/Nginx
  - topic/计算机网络
  - topic/MOC
status: to-review
aliases:
  - Nginx 知识地图
  - Nginx MOC
---

# Nginx 知识点汇总 — 知识地图

> [!abstract] 本笔记为 MOC（Map of Content）
> Nginx 是常用的反向代理服务器，涵盖请求路由、代理转发、超时配置、安全限制、日志记录等核心功能。本 MOC 汇总相关的原子笔记。

## location 匹配与路由

| 主题 | 核心笔记 |
|:-----|:---------|
| **location 匹配优先级** | [[Nginx-location块]] |
| **try_files 请求尝试** | [[Nginx-try-files]] |
| **proxy_pass 路径处理** | [[Nginx-proxy-pass-路径处理]] |
| **alias 目录访问与 301 重定向** | [[Nginx-alias-301-redirect]] |

## 反向代理与转发

| 主题 | 核心笔记 |
|:-----|:---------|
| **proxy_pass 斜杠行为** | [[Nginx-proxy-pass-路径处理]] |
| **proxy_read_timeout 超时** | [[Nginx-proxy-read-timeout]] |
| **proxy_redirect 重写 Location** | [[Nginx-proxy-redirect]] |
| **案例：proxy_pass / proxy_redirect 配置调整** | [[案例-Nginx-proxy_pass-proxy_redirect配置调整]] |

## 正则与重写

| 主题 | 核心笔记 |
|:-----|:---------|
| **Nginx 正则表达式语法** | [[Nginx-正则表达式]] |

## 重定向控制

| 主题 | 核心笔记 |
|:-----|:---------|
| **server_name_in_redirect / port_in_redirect** | [[Nginx-server_name_in_redirect-port_in_redirect]] |
| **proxy_set_header 控制 Location** | [[Nginx-proxy_set_header-redirect-location]] |

## 请求限制与安全

| 主题 | 核心笔记 |
|:-----|:---------|
| **客户端请求体大小限制** | [[Nginx-client-max-body-size]] |
| **请求体存储缓冲区大小** | [[Nginx-client-body-buffer-size]] |
| **CORS 跨域资源共享配置** | [[Nginx-CORS配置]] |
| **请求头支持下划线** | [[Nginx-underscores-in-headers]] |

## 连接与性能

| 主题 | 核心笔记 |
|:-----|:---------|
| **keepalive 超时配置** | [[Nginx-keepalive-timeout]] |

## 日志

| 主题 | 核心笔记 |
|:-----|:---------|
| **日志格式定义（log_format）** | [[Nginx-log-format]] |
| **日志配置** | [[Nginx-日志配置]] |

## 相关原子笔记

```dataview
TABLE
  file.tags as "标签"
FROM "10-Topics"
WHERE file.name IN ["Nginx-location块", "Nginx-try-files", "Nginx-proxy-pass-路径处理", "Nginx-proxy-read-timeout", "Nginx-proxy-redirect", "Nginx-正则表达式", "Nginx-client-max-body-size", "Nginx-client-body-buffer-size", "Nginx-CORS配置", "Nginx-underscores-in-headers", "Nginx-keepalive-timeout", "Nginx-log-format", "Nginx-日志配置", "Nginx-alias-301-redirect", "Nginx-server_name_in_redirect-port_in_redirect", "Nginx-proxy_set_header-redirect-location", "案例-Nginx-proxy_pass-proxy_redirect配置调整"]
SORT file.name ASC
```

## 外部关联

- [[四层与七层负载均衡对比]] — Nginx 作为七层负载均衡的核心应用
- [[Helm-Chart模板概述]] — Nginx 常通过 Helm Chart 在 K8s 中部署
