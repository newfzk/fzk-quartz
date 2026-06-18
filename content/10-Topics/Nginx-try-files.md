---
title: nginx请求尝试配置
type: basic-note
date: 2025-05-29
tags:
  - topic/Nginx
  - topic/计算机网络
status: to-review
---

# nginx请求尝试配置

`try_files` `http_core`核心模块提供的指令，可将请求按照指定顺序依次尝试解析

语法规则：

```nginx
格式一：
try_files file ... uri;
或 格式二：
try_files file ... =code;
```

可用上下文：`server`, `location`

- 按照指定的file顺序查找存在的文件 并使用找到的第一个文件进行请求处理；
- 查找路径是按照给定的`root`或`alias`为根路径来查找的；
- 如果给定的file都没有匹配到，则重新请求最后一个`uri`或返回格式二中的`code`

使用示例：

```nginx
# 文件 、 主页 、 404
try_files $uri /index.html =404;
# 文件 目录 404
try_files $uri $uri/ =404;
```

## 相关笔记

- [[Nginx-location块]]