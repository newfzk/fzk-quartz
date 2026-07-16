---
title: Nginx proxy_redirect — 重写后端返回的 Location 头
type: basic-note
date: 2026-07-10
tags:
  - topic/Nginx
  - topic/计算机网络
status: to-review
---

# Nginx proxy_redirect — 重写后端返回的 Location 头

`proxy_redirect` 用于修改后端服务器返回的 `Location` 响应头（即 301/302 重定向地址），使其适应客户端的访问路径。

## 为什么需要 proxy_redirect

当 Nginx 作为反向代理时，后端返回的重定向地址通常是内部地址（如 `http://192.168.1.1/apps/...`），这个地址客户端无法直接访问。`proxy_redirect` 可以将内部地址改写为客户端可访问的外部地址。

## 语法

```nginx
proxy_redirect 原始地址  替换地址;
proxy_redirect ~正则匹配  替换地址;
```

## 常见场景

场景：通过 `http://external:30331/rs/...` 访问后端，后端返回的 `Location` 是 `http://internal/apps/...`。

### 方案一：精确替换（硬编码）

```nginx
location ^~ /rs {
    proxy_pass http://nginx-svc:81/;

    # 将后端返回的绝对地址改写为外部可访问地址（含端口和 /rs 前缀）
    proxy_redirect ~^http://192\.168\.168\.161/apps/(.*) http://192.168.168.161:30331/rs/apps/$1;
}
```

### 方案二：通用正则替换（推荐）

将任意绝对 URL 重写为相对路径，让浏览器基于当前请求的 `host:port` 自动拼接：

```nginx
location ^~ /rs {
    proxy_pass http://nginx-svc:81/;

    # 将后端返回的绝对 URL 转为相对路径，并加上 /rs 前缀
    # 正则详解 → [[Nginx-正则表达式#实战拆解proxy_redirect中的正则]]
    proxy_redirect ~^https?://[^/]+(/.*)$ /rs$1;
}
```

> 正则详解 → [[Nginx-正则表达式#实战拆解proxy_redirect中的正则]]

或使用 `$http_host` 变量构造完整 URL：

```nginx
proxy_redirect ~^https?://[^/]+(/.*)$ $scheme://$http_host/rs$1;
```

### 方案三：配合默认规则显式覆盖

当 Nginx 默认的 `proxy_redirect default` 可能与自定义规则冲突时，显式声明：

```nginx
location ^~ /rs {
    proxy_pass http://nginx-svc:81/;

    proxy_redirect default;
    proxy_redirect ~^https?://[^/]+(/.*)$ /rs$1;   # 自定义规则覆盖默认
}
```

## 关键变量

| 变量 | 含义 | 示例值 |
|:---|:---|:---:|
| `$scheme` | 请求协议 | `http` / `https` |
| `$http_host` | 客户端请求的 Host 头（含端口） | `192.168.168.161:30331` |
| `$host` | 客户端请求的 Host（不含端口） | `192.168.168.161` |

## 排错提示

如果 `proxy_redirect` 未生效，检查：

1. 确认 `proxy_redirect` 在 `location` 块内且被加载（可用 `add_header X-Test "loaded";` 临时验证）
2. 默认的 `proxy_redirect default` 可能与自定义规则冲突，需要显式覆盖
3. 查看 Nginx 错误日志 `/var/log/nginx/error.log`

## 相关笔记

- [[Nginx-proxy-pass-路径处理]]
- [[Nginx-location块]]
