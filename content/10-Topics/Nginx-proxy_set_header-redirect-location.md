---
title: proxy_set_header 控制 redirect 后的 Location — Host、Real-IP、Proto
type: basic-note
date: 2026-07-14
tags:
  - topic/Nginx
  - topic/计算机网络
status: to-review
---

# proxy_set_header 控制 redirect 后的 Location — Host、Real-IP、Proto

当设置 `server_name_in_redirect off;` 和 `port_in_redirect off;` 后，Nginx 生成重定向 URL 时会使用请求头中的 `Host` 字段来构造 `Location`。此时可以通过 `proxy_set_header` **主动控制请求头**，进而影响 Nginx 最终生成的重定向地址。

## 核心原理

Nginx 内部生成重定向的 `Location` 头构造逻辑：

```
Location = scheme + "://" + Host头中的值 + 路径
```

控制链路：
```
proxy_set_header Host xxx  →  请求的 Host 头  →  server_name_in_redirect off
                                                       ↓
                                              Location 头中的域名
```

## 常用 proxy_set_header 配置

```nginx
location / {
    proxy_pass http://backend;

    # 控制 Host（影响 server_name_in_redirect/port_in_redirect 生成的 Location）
    proxy_set_header Host $host;                    # 仅域名，无端口
    proxy_set_header Host $host:$server_port;       # 域名+端口

    # 传递客户端真实 IP
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

    # 传递请求协议（http/https）
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

## 关键变量说明

| 变量 | 含义 | 示例值 |
|:----|:-----|:------:|
| `$host` | 请求 Host 头（不含端口） | `example.com` |
| `$http_host` | 请求 Host 头（含端口，原样） | `example.com:8080` |
| `$host:$server_port` | 域名 + Nginx 监听端口 | `example.com:8080` |
| `$scheme` | 请求协议 | `http` / `https` |
| `$remote_addr` | 客户端 IP | `10.0.0.1` |
| `$proxy_add_x_forwarded_for` | 在原有 `X-Forwarded-For` 头后追加客户端 IP，已有值时用逗号分隔追加，无值时等于 `$remote_addr` | `10.0.0.1` / `192.168.1.1, 10.0.0.1` |

## 典型场景

### 场景一：保持客户端 Host 到后端，同时保证重定向地址正确

```nginx
server {
    listen 8080;
    server_name api.example.com;

    server_name_in_redirect off;  # 使用 Host 头决定重定向域名
    port_in_redirect off;         # 外部标准端口无需显示

    location / {
        proxy_pass http://backend:3000;

        # 透传客户端 Host，后端和 Nginx 重定向都基于此
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

> 请求 `http://api.example.com/api/login` → Nginx 返回 301 到 `http://api.example.com/api/login/`（Host 正确，无端口）

### 场景二：LB 终止 TLS，后端需要知道原始协议

```nginx
server {
    listen 80;
    server_name myapp.com;

    server_name_in_redirect off;
    port_in_redirect off;

    location / {
        proxy_pass http://backend;

        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto https;  # LB 终止了 TLS，告知后端原始协议是 https
        proxy_set_header X-Forwarded-Port 443;
    }
}
```

### 场景三：自定义 Host 覆盖（调试或特殊路由）

```nginx
location / {
    proxy_pass http://backend;

    # 强制覆盖 Host 为特定值（慎重使用）
    proxy_set_header Host "custom-domain.com";
}
```

## 与 proxy_redirect 的关系

- `server_name_in_redirect` / `port_in_redirect` + `proxy_set_header Host` 控制的是 **Nginx 自身生成的重定向**（如 `return 301`、`rewrite ... redirect`）。
- [[Nginx-proxy-redirect]] 控制的是 **后端返回的重定向**的 `Location` 改写。
- 两者可配合使用：上游关闭 `server_name_in_redirect` 透传客户端 Host，下游用 `proxy_redirect` 修正后端内部地址。

## 排错提示

1. **查看实际返回的 Location**：`curl -I http://your-domain/path`
2. **确认 server_name_in_redirect 为 off**：否则 Nginx 不会使用 Host 头，而是用 `server_name`
3. **区分 `$host` 与 `$http_host`**：前者不含端口，后者包含原始端口
4. **X-Forwarded-Proto 影响后端协议判断**：如果后端根据此头生成重定向 URL，需确保其值正确

## 相关笔记

- [[Nginx-server_name_in_redirect-port_in_redirect]] — 上游控制指令详解
- [[Nginx-proxy-redirect]] — 后端 Location 改写
- [[Nginx-proxy-pass-路径处理]] — proxy_pass 路径转发行为
