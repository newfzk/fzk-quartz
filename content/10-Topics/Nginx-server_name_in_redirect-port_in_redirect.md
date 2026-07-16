---
title: Nginx server_name_in_redirect 与 port_in_redirect — 控制重定向 URL 的 Host 与端口
type: basic-note
date: 2026-07-14
tags:
  - topic/Nginx
  - topic/计算机网络
status: to-review
---

# Nginx `server_name_in_redirect` 与 `port_in_redirect` — 控制重定向 URL 的 Host 与端口

这两个指令控制 Nginx **在内部生成重定向 URL**（即 `Location` 响应头）时，使用哪些信息来构造重定向地址。它们影响的是 **Nginx 自身发出的重定向**（如 `return 301`、`rewrite ... redirect`、`alias` 自动补斜杠等），而非代理后端返回的重定向。

## server_name_in_redirect

控制 Nginx 在 `Location` 头中使用 `server_name` 还是请求的 `Host` 头。

| 取值 | 行为 | 示例 `Location` |
|:---:|:-----|:---------------:|
| `on`（默认） | 使用 `server_name` 指令的值 | `http://my-server.com/path/` |
| `off` | 使用请求头中的 `Host` 字段 | `http://example.com/path/` |

```nginx
server {
    listen 80;
    server_name my-server.com;

    server_name_in_redirect off;  # 使用客户端请求的 Host
    # server_name_in_redirect on; # 使用 my-server.com

    location = /api {
        return 301 /api/;
    }
}
```

> [!tip] 使用场景
> 当 Nginx 背后有负载均衡器（LB）或 API 网关时，客户端请求的 `Host` 可能是网关的域名，而 `server_name` 是内部域名。此时需要 `server_name_in_redirect off`，让客户端看到的是**可访问的外部域名**。

## port_in_redirect

控制 Nginx 在 `Location` 头中是否包含端口号。

| 取值 | 行为 | 示例 `Location` |
|:---:|:-----|:---------------:|
| `on`（默认） | 包含监听端口 | `http://example.com:8080/path/` |
| `off` | 省略端口（仅当端口为 80/443 时省略） | `http://example.com/path/` |

```nginx
server {
    listen 8080;
    server_name example.com;

    port_in_redirect off;  # Location 中不包含 :8080
}
```

> [!warning] 非标准端口
> 如果 Nginx 监听的是非标准端口（如 8080、8443），`port_in_redirect off` 会导致客户端使用默认端口（80/443）重试，可能无法访问。此时应保持 `port_in_redirect on`，或通过 `proxy_set_header Host $host:$server_port` 传递完整 Host。

## 典型配合场景

### 场景：Nginx 在 LB 之后，需要保留客户端 Host

```nginx
server {
    listen 80;
    server_name internal-service;

    server_name_in_redirect off;  # 保留客户端请求的域名
    port_in_redirect off;         # 外部端口是标准 80/443，无需显示端口

    location / {
        proxy_pass http://backend:8080;
    }
}
```

### 场景：Nginx 直接对外服务，非标准端口

```nginx
server {
    listen 8443 ssl;
    server_name api.example.com;

    server_name_in_redirect on;   # 使用配置的 server_name
    port_in_redirect on;          # 保留 8443 端口，避免客户端用 443 重试

    location / {
        proxy_pass http://backend:8080;
    }
}
```

## 排错提示

如果浏览器访问时被重定向到了错误的地址：

1. 检查 Nginx 返回的 `Location` 头：`curl -I http://your-domain/path`
2. 确认 `server_name_in_redirect` 的取值——`on` 使用 `server_name`，`off` 使用 `Host` 头
3. 确认 `port_in_redirect` 的取值——非标准端口下关闭可能导致访问失败
4. 这些指令仅影响 Nginx **自身生成**的重定向，不影响 [[Nginx-proxy-redirect]] 处理的代理后端返回的重定向

## 相关笔记

- [[Nginx-proxy-redirect]] — 代理后端返回的重定向处理
- [[Nginx-proxy_set_header-redirect-location]] — 通过 proxy_set_header 控制 Location
- [[Nginx-alias-301-redirect]] — alias 目录访问自动补斜杠的重定向
