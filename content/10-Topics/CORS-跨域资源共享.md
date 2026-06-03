---
title: CORS 跨域资源共享
type: basic-note
date: 2025-05-28
tags: cors, HTTP, 浏览器, 跨域
---

# CORS 跨域资源共享

CORS（Cross-Origin Resource Sharing，跨域资源共享）是一种 Web 浏览器用于限制跨域 HTTP 请求的机制，通过 HTTP 标头来控制跨域请求的访问。

> **跨域**：一个域名的网页向另一个域名的服务器发送请求。

## 同源策略

浏览器默认**不允许**从一个域名的网页向另一个域名的服务器发送请求。这是浏览器的安全策略。

## CORS 主要作用

- 控制跨域请求：哪些域名、哪些请求方法允许
- 保护资源：避免网页脚本恶意访问其他域名资源
- 安全标准：确保跨域请求安全可控

## 关键 HTTP 标头

| 标头 | 作用 |
|------|------|
| `Access-Control-Allow-Origin` | 指定允许的域名 |
| `Access-Control-Allow-Methods` | 允许的 HTTP 方法（GET, POST, PUT 等） |
| `Access-Control-Allow-Headers` | 允许的请求头 |
| `Access-Control-Expose-Headers` | 允许暴露的响应头 |
| `Access-Control-Max-Age` | 预检请求有效期 |
| `Access-Control-Allow-Credentials` | 是否允许携带 Cookie |

## 简单请求 vs 预检请求

- **简单请求**：不会触发 CORS 预检请求（如 GET、POST 且 Content-Type 为表单类型）
- **非简单请求**：浏览器先发送 OPTIONS 预检请求，确认服务器允许后再发送实际请求

## 相关笔记

- [[Nginx-CORS配置]]

## 参考

- [MDN Web Docs - CORS](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Guides/CORS)