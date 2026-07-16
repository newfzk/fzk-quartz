---
title: Nginx proxy_pass 路径处理 — 尾部斜杠的关键影响
type: basic-note
date: 2026-07-10
tags:
  - topic/Nginx
  - topic/计算机网络
status: reviewed
---

# Nginx proxy_pass 路径处理 — 尾部斜杠的关键影响

`proxy_pass` 指令中**是否以斜杠结尾**，决定了 Nginx 转发请求时是否丢弃 `location` 匹配的前缀，这是配置反向代理时最容易踩的坑之一。

## 核心规则

| proxy_pass 写法 | 转发行为 | 示例 |
|:---:|:---|:---|
| **无 URI**（无尾部斜杠/路径） | 将原始请求的**完整 URI** 原样转发给后端 | `location /rs { proxy_pass http://backend; }` <br> 请求 `/rs/apps/sys` → 后端收到 `/rs/apps/sys` |
| **有 URI**（有尾部斜杠或路径） | 将 `location` 匹配到的部分**替换为** `proxy_pass` 中的 URI | `location /rs { proxy_pass http://backend/; }` <br> 请求 `/rs/apps/sys` → 后端收到 `/apps/sys` |

## 判断方法

关键在于 `proxy_pass` 指令的**值中是否包含 URI 部分**（不一定是斜杠，任何路径都算）：

```nginx
# 无 URI → 原样传递
proxy_pass http://backend;            # 没有 /
proxy_pass http://backend;            # 没有路径

# 有 URI → 替换匹配前缀
proxy_pass http://backend/;           # 有 /
proxy_pass http://backend/api;        # 有 /api
proxy_pass http://backend/api/;       # 有 /api/
```

## 实际案例

场景：通过 `/rs` 前缀访问后端服务，但希望后端收到时不带 `/rs` 前缀。

```nginx
# ❌ 错误：后端收到 /rs/apps/sys/sys0020（带了 /rs 前缀）
location ^~ /rs {
    proxy_pass http://nginx-svc:81;
}

# ✅ 正确：后端收到 /apps/sys/sys0020（/rs 被替换为 /）
location ^~ /rs {
    proxy_pass http://nginx-svc:81/;
}
```

## 替代方案

如果不希望修改 `proxy_pass`，也可以使用 `rewrite` 显式重写路径：

```nginx
location ^~ /rs {
    rewrite ^/rs(.*)$ $1 break;
    proxy_pass http://nginx-svc:81;
}
```

使用带斜杠的 `proxy_pass` 更简洁且为官方推荐做法。

## 相关笔记

- [[Nginx-location块]]
- [[Nginx-proxy-redirect]]
