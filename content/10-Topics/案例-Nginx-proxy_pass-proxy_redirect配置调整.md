---
tags:
  - topic/Nginx
  - topic/案例
  - topic/计算机网络
status: to-review
---

# 案例：Nginx proxy_pass / proxy_redirect 配置调整

实际项目中 Nginx 反向代理配置调整记录，涉及两个问题。

## 场景

通过 `http://外部地址:30331/rs/apps/sys/sys0020` 访问后端服务，Nginx 将请求反向代理到内部 `nginx-svc:81`。

## 问题一：路径前缀处理（proxy_pass）

**需求**：访问 `/rs/...` 时，后端收到 `/apps/...`（去掉 `/rs` 前缀）。

**初始配置**（后端原样收到 `/rs/...`）：
```nginx
location ^~ /rs {
    proxy_pass http://nginx-svc:81;    # 无尾部斜杠
}
```

**修复**：在 `proxy_pass` 后添加斜杠，触发路径替换：
```nginx
location ^~ /rs {
    proxy_pass http://nginx-svc:81/;   # 有尾部斜杠
}
```

> 原理参见 [[Nginx-proxy_pass-路径处理]]：`proxy_pass` 有 URI 时，替换 location 匹配到的前缀。

## 问题二：Location 头重写（proxy_redirect）

**需求**：后端返回 `Location: http://192.168.168.161/apps/...`，需重写为外部可访问的地址（含端口和 `/rs` 前缀）。

**解决方案**：用通用正则将绝对 URL 转为相对路径：

```nginx
location ^~ /rs {
    proxy_pass http://nginx-svc:81/;
    proxy_read_timeout 300s;

    # 将后端返回的绝对 URL 转为相对路径，并加上 /rs 前缀
    proxy_redirect default;
    proxy_redirect ~^https?://[^/]+(/.*)$ /rs$1;
}
```

也可使用 `$http_host` 变量构造完整 URL：
```nginx
proxy_redirect ~^https?://[^/]+(/.*)$ $scheme://$http_host/rs$1;
```

> 原理参见 [[Nginx-proxy-redirect]]。

## 排错记录

配置 `proxy_redirect` 时发现自定义规则未生效，原因：
1. Nginx 默认的 `proxy_redirect default` 可能与自定义规则冲突
2. 需要在自定义规则前先 `proxy_redirect default;` 显式启用默认规则，再用自定义规则覆盖

## 最终配置

```nginx
location ^~ /rs {
    proxy_pass http://nginx-svc:81/;
    proxy_read_timeout 300s;
    proxy_set_header Host $http_host;

    proxy_redirect default;
    proxy_redirect ~^https?://[^/]+(/.*)$ /rs$1;
}
```

## 相关笔记

- [[Nginx-proxy_pass-路径处理]] — proxy_pass 尾部斜杠的路径替换规则
- [[Nginx-proxy-redirect]] — proxy_redirect 重写后端 Location 头
- [[Nginx-location块]]
