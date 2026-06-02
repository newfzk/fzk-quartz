---
title: 跨域资源共享
type: basic-note
date: 2025-05-28
tags: cors, concept
---

# 跨域资源共享 CORS

CORS 中文 跨域资源共享 一种Web浏览器用于限制、约束跨域`HTTP`请求的机制,
通过HTTP标头来控制跨域请求的访问。

> 跨域 跨域域名 一个域名的网页向另一个域名的服务器发送请求

浏览器遵循同源策略，
**默认不允许从一个域名的网页向另一个域名的服务器发送请求**。

如果一定需要跨域资源访问，则需要依赖CORS 主要作用如下：

- 控制跨域请求：如哪些域名、哪些请求方法等
- 保护资源：避免网页的脚本恶意访问其他域名的资源
- 安全标准：浏览器使用CORS来确保跨域请求的安全和可控

CORS通过`HTTP标头 header`字段进行跨域访问权限声明 主要设计标头如下：

- `Access-Control-Allow-Origin`: 指定哪些域名可以访问资源。
- `Access-Control-Allow-Methods`: 指定允许使用的HTTP 方法，例如GET, POST, PUT。
- `Access-Control-Allow-Headers`: 指定允许的请求头。
- `Access-Control-Expose-Headers`: 指定返回的响应头。
- `Access-Control-Max-Age`: 指定预检请求的有效期。
- `Access-Control-Allow-Credentials`: 指定是否允许携带Cookie。

TODO 看不太懂了 后面再说

- [简单请求](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Guides/CORS#%E7%AE%80%E5%8D%95%E8%AF%B7%E6%B1%82): **不会**触发CORS预检请求

## 相关参考

- mdn web docs <https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Guides/CORS>
- 维基百科 <https://zh.wikipedia.org/zh-cn/%E8%B7%A8%E4%BE%86%E6%BA%90%E8%B3%87%E6%BA%90%E5%85%B1%E4%BA%AB>
