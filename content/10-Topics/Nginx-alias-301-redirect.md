---
title: Nginx alias 目录访问引发的 301 重定向
type: basic-note
date: 2026-07-14
tags:
  - topic/Nginx
  - topic/计算机网络
status: to-review
---

# Nginx alias 目录访问引发的 301 重定向

使用 `alias` 指令将请求映射到文件系统目录时，若请求 URI **末尾没有斜杠**，Nginx 会返回 **301 重定向**，要求客户端加上尾部斜杠重新请求。

## 现象

```nginx
location /download {
    alias /var/www/files;
}
```

请求 `GET /download` → Nginx 返回 `301`，`Location: /download/`。

浏览器自动跟随重定向，再次请求 `/download/` → 正常返回目录索引或文件。

## 根因

- `alias` 将 URI 映射到文件系统路径后，Nginx 需要判断该路径指向的是**文件**还是**目录**。
- 若请求 URI 末尾**没有斜杠**，而 `alias` 指向的路径是**目录**，Nginx 无法确定客户端意图是访问目录本身（展示索引）还是访问同名文件。
- 因此 Nginx **自动补齐斜杠**，返回 301 重定向到带斜杠的 URI，再正常处理。

> 这与 `root` 指令的行为一致——Nginx 对目录访问的规范行为就是需要尾部斜杠。

## 影响

1. **对 API 类请求不友好**：若 API 路径恰好匹配了 alias 目录，多余的重定向会增加一次 RTT。
2. **反向代理场景需注意**：Nginx 作为反向代理时，这种重定向可能导致客户端看到的地址与预期不符。
3. **配合 `proxy_redirect` 使用时**：若后端也返回重定向，多个重定向叠加会增加延迟。

## 配置建议

### 方案一：使用 `root` 替代 `alias`（如果适用）

```nginx
location /download/ {
    root /var/www;
}
```

- `root` 直接将 URI 拼接到 root 路径后，不存在 alias 的映射歧义。
- 限制：只能用于路径前缀匹配（location 末尾带斜杠），无法像 `alias` 那样自由重映射路径。

### 方案二：显式 rewrite，避免 Nginx 自动判断

```nginx
location /download {
    rewrite ^/download(/.*)?$ /download$1 permanent;
}
```

### 方案三：在 location 中显式处理尾部斜杠

```nginx
location = /download {
    return 301 /download/;
}

location /download/ {
    alias /var/www/files/;
}
```

### 方案四：关闭目录索引，避免歧义

若 alias 目录不需要展示目录列表，也可通过 `autoindex off` 确认行为，但 301 重定向仍会发生——该重定向发生在索引判断之前。

## 相关笔记

- [[Nginx-location块]] — location 匹配规则
- [[Nginx-proxy-redirect]] — 代理场景下重定向处理
- [[Nginx-proxy-pass-路径处理]] — proxy_pass 尾部斜杠的关键影响
