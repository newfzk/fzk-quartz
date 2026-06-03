---
title: nginx-conf-proxy-read-timeout
type: basic-note
date: 2025-10-24
tags: nginx, conf, timeout
---

# nginx-conf-proxy-read-timeout

nginx超时配置-后端请求最大等待时间(作用于Nginx到上游服务器-后端应用服务器的连接)

这个指令设置了 Nginx 等待从后端服务器读取响应数据的最大时间。如果在这个时间内后端没有返回任何数据，Nginx 将关闭与客户端的连接，并返回一个 504 Gateway Time-out 错误。

配置示例：

```conf
location / {
    proxy_pass http://my_backend;
    # 设置等待后端响应的超时时间为 60 秒
    proxy_read_timeout 60s;
}
```

相关报错日志（超出`proxy_read_timeout`后）

```log
2025/10/24 14:45:56 [error] 30#30: *39 upstream timed out (110: Operation timed out) while reading response header from upstream, client: 10.21.40.235, server: extjs-ui, request: "POST /mrp/mrp20/mrp2011/makePlan HTTP/1.1", upstream: "http://10.68.204.238:80/mrp/mrp20/mrp2011/makePlan", host: "10.20.21.10:30321", referrer: "http://10.20.21.10:30321/"
```

## 相关笔记

- [[Nginx-keepalive-timeout]]