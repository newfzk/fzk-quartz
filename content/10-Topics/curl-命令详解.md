---
title: curl 命令详解
created: 2026-06-11
tags:
  - topic/Linux
  - topic/Linux/命令
  - topic/计算机网络
status: to-review
---

# curl 命令详解

## 简介

`curl` 是一个基于 **URL 语法** 的命令行数据传输工具，支持 HTTP/HTTPS/FTP/TELNET 等多种协议。是最常用的 API 调试和网络排查工具之一。

## 命令解析：端口可达性测试

```bash
curl -v --connect-timeout 3 telnet://172.20.0.7:53
```

| 部分 | 含义 |
|------|------|
| `curl` | 基础命令 |
| `-v` (--verbose) | 输出详细连接过程（DNS 解析、TCP 握手、TLS 握手、请求/响应头） |
| `--connect-timeout 3` | 连接超时 **3 秒** |
| `telnet://172.20.0.7:53` | 使用 `telnet` 协议建立裸 TCP 连接到 `172.20.0.7:53`（DNS 端口） |

> 此命令等价于 `nc -zv -w 3 172.20.0.7 53`，都用于测试 TCP 端口是否开放。参阅 [[nc-netcat命令详解]]。

## 常用参数

### 连接控制

| 参数 | 说明 | 示例 |
|------|------|------|
| `--connect-timeout <秒>` | 连接超时 | `--connect-timeout 5` |
| `--max-time <秒>` | 整个传输最大时间 | `--max-time 30` |
| `--retry <次数>` | 失败重试次数 | `--retry 3` |
| `-x/--proxy <地址>` | 使用代理 | `-x http://127.0.0.1:7890` |

### 输出与调试

| 参数 | 说明 | 示例 |
|------|------|------|
| `-v` | 详细模式 | `curl -v https://example.com` |
| `-i` | 输出含响应头 | `curl -i https://api.example.com` |
| `-I` | 只发 HEAD 请求 | `curl -I https://example.com` |
| `-s` | 静默模式（不显示进度/错误） | `curl -s https://api.example.com` |
| `-S` | 与 `-s` 配合只显错误 | `curl -sS https://api.example.com` |
| `-w/--write-out` | 输出格式化统计信息 | `curl -w "\n%{http_code}\n" url` |

### 请求控制

| 参数 | 说明 | 示例 |
|------|------|------|
| `-X/--request <方法>` | 指定 HTTP 方法 | `-X POST` / `-X PUT` |
| `-H/--header` | 添加请求头 | `-H "Authorization: Bearer xxx"` |
| `-d/--data` | 发送 POST 数据 | `-d '{"key":"value"}'` |
| `-F/--form` | 模拟表单提交 | `-F "file=@photo.jpg"` |
| `-b/--cookie` | 发送 Cookie | `-b "session=abc123"` |
| `-c/--cookie-jar` | 保存返回的 Cookie | `-c cookies.txt` |
| `-A/--user-agent` | 自定义 UA | `-A "Mozilla/5.0 ..."` |

### 认证与安全

| 参数 | 说明 | 示例 |
|------|------|------|
| `-u/--user` | HTTP 基本认证 | `-u admin:password` |
| `-k/--insecure` | 跳过 TLS 验证 | `-k https://self-signed.local` |
| `--cacert` | 指定 CA 证书 | `--cacert /path/ca.pem` |
| `--cert` | 客户端证书 | `--cert client.p12:password` |

### 文件操作

| 参数 | 说明 | 示例 |
|------|------|------|
| `-o/--output` | 保存到文件 | `-o file.zip` |
| `-O` | 用 URL 文件名保存 | `-O https://example.com/file.zip` |
| `-C/--continue-at` | 断点续传 | `-C - -O https://example.com/bigfile.zip` |
| `--limit-rate` | 限速 | `--limit-rate 100K` |

## 核心调试指标（`-w`）

```bash
curl -w "\n\
  time_namelookup:   %{time_namelookup}s\n\
  time_connect:      %{time_connect}s\n\
  time_appconnect:   %{time_appconnect}s\n\
  time_starttransfer: %{time_starttransfer}s\n\
  time_total:        %{time_total}s\n" \
  -o /dev/null -s https://example.com
```

| 变量 | 含义 | 排查方向 |
|------|------|---------|
| `time_namelookup` | DNS 解析耗时 | DNS 服务器慢 / 配置问题 |
| `time_connect` | TCP 三次握手耗时 | 网络延迟 / 防火墙 |
| `time_appconnect` | TLS 握手耗时（HTTPS） | 证书链 / 加密协商 |
| `time_starttransfer` | 首字节到达时间（TTFB） | 服务器端性能 |
| `time_total` | 总耗时 | 综合评估 |

## `telnet://` vs `http://` 的区别

| | `telnet://host:port` | `http://host:port` |
|---|---|---|
| **行为** | 只建 TCP 连接，不发数据 | 发送完整 HTTP 请求 |
| **用途** | 测试端口可达性 | 测试 HTTP 服务本身 |
| **退出方式** | 等待 stdin 输入，需手动退出 | 请求完成后自动退出 |

## 常见场景

### API 调试

```bash
# GET + 查看响应头
curl -i https://api.github.com/users/octocat

# POST JSON
curl -X POST https://api.example.com/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"123456"}'

# 仅看状态码
curl -s -o /dev/null -w "%{http_code}" https://example.com
```

### 网络排查

```bash
# 端口可达性测试
curl -v --connect-timeout 3 telnet://172.20.0.7:53

# 测试代理
curl -v -x http://127.0.0.1:7890 https://www.google.com

# 接口性能分析（配合 -w）
curl -s -o /dev/null -w "DNS: %{time_namelookup}s | TCP: %{time_connect}s | TTFB: %{time_starttransfer}s | Total: %{time_total}s\n" https://api.example.com/health
```

### 文件操作

```bash
# 断点续传
curl -C - -O https://releases.ubuntu.com/22.04/ubuntu-22.04-desktop-amd64.iso

# 限速下载
curl --limit-rate 500K -O https://example.com/bigfile.zip

# 上传文件
curl -F "file=@/path/to/photo.jpg" -F "description=vacation" \
  https://api.example.com/upload
```

## 相关笔记

- [[nc-netcat命令详解]] — nc 同样是 TCP 端口测试工具，`nc -zv -w 3 host port` 等价于 `curl telnet://host:port`
- [[tcpdump-常见场景示例]] — 抓包分析网络问题
- [[tcpdump-过滤表达式]] — tcpdump 过滤语法
- [[接口性能排查指南]] — 结合 curl 的 `-w` 指标进行 API 性能分析
- [[iptables详解]] — 防火墙规则可能影响 curl 连接
