---
title: nginx配置-请求头可带下划线
type: basic-note
date: 2025-05-29
tags: nginx, conf, http, header, underscore, underline
---

# nginx配置-请求头可带下划线

nginx 默认忽略带下划线的header 如果http头中有带有下划线的头 需要进行如下配置：

```nginx
underscores_in_headers on;
```

标准的HTTP头应使用**减号**拼接

## 参考链接

- 原因：避免headers映射为CGI变量时出现歧义 <https://github.com/cherishman2005/nginx-modules/blob/master/nginx header头字段尽量不要使用下划线.md>
- nginx 0.7 changelog <http://nginx.org/en/CHANGES-0.7>
