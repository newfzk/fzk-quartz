---
title: nginx跨域资源共享配置
type: basic-note
date: 2025-05-28
tags: nginx, conf, cors
---

# nginx跨域资源共享配置

**错误的解决方案!**

本次遇到的问题是前台访问nginx nginx将请求代理转发给后台 后台重定向 浏览器向重定向后的新路径（新ip）发请求

因为 nginx 不在新路径侧 所以修改nginx配置无效！ 最后通过 location 代码块的反向代理解决跨域

- 跨域资源共享 [[CORS-跨域资源共享]]
- nginx官方文档 <https://docs.nginx.com/nginx-management-suite/acm/how-to/policies/cors/>

通过一下配置进行nginx跨域资源共享配置：

```shell
# 允许跨域请求的域名（改成你的前端地址或通配符 *）
add_header 'Access-Control-Allow-Origin' 'http://192.168.168.176:30321' always;

# 允许的请求方法
add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS, PUT, DELETE' always;

# 允许的请求头
add_header 'Access-Control-Allow-Headers' 'Content-Type, Authorization, X-Requested-With' always;

# 允许携带凭证（如 cookies）
add_header 'Access-Control-Allow-Credentials' 'true' always;

# 预检请求缓存时间
add_header 'Access-Control-Max-Age' 1728000 always;
```

## 相关笔记

- [[Nginx-location块]]