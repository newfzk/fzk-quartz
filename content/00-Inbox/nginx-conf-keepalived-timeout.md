---
title: nginx-conf-keepalived-timeout
type: basic-note
date: 2025-10-24
tags: nginx, conf, timeout
---

# nginx-conf-keepalived-timeout

nginx超时配置-空闲连接保持时间

> - 减少TCP连接建立和关闭的次数，提高响应速度；
> - 及时关闭空闲连接，释放服务器资源；

```conf
http {
    # 设置保持连接的超时时间为 65 秒
    keepalive_timeout 65;
}
```
