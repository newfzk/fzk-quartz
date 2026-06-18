---
title: Nginx-请求体存储缓冲区大小配置
type: basic-note
date: 2025-08-26
tags:
  - topic/Nginx
  - topic/计算机网络
status: to-review
---

# Nginx-请求体存储缓冲区大小配置

`client_body_buffer_size`定义了读取请求体时使用的缓冲区大小, 对于大文件上传，适当调整可能有助于性能

配置示例：`client_body_buffer_size 256k;` （默认16k）

nginx处理请求的流程大致如下：

1. 接收数据：Nginx从网络连接中，一块块的读取数据
2. 缓存数据：读取的数据先临时存放在内存中，即缓冲区`Buffer`
3. 处理或传递数据：
   1. 数据一次性接收完毕（请求体大小 <= `client_body_buffer_size`） 只涉及内存读写，速度快，效率高，cpu友好
   2. 数据分批次接收（请求体大小 > `client_body_buffer_size`） 内存+磁盘读写（内存写满，转存至临时磁盘文件，继续内存接收、转存，直至请求头数据接收完毕），最终体现为**一部分内存数据+一个或多个磁盘临时文件**

设计原因：出于权衡策略：

1. 保护服务器内存：避免恶意攻击导致服务器内存占满；
2. 可靠性保证：及时将内存数据持久化到磁盘，避免长连接持续占用大量内存；
3. 灵活调整：可以根据服务器情况、使用场景，灵活调整`client_body_buffer_size`，寻找到适合自己的最佳平衡；

配置推荐：`client_body_buffer_size 256k;`

## 相关笔记

- [[Nginx-client-max-body-size]]