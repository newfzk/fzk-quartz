---
title: SSH 会话保活与超时断连排查
date: 2026-06-24
tags:
  - topic/Linux
  - topic/计算机网络
aliases:
  - SSH 保活
  - ClientAliveInterval
  - ServerAliveInterval
  - TMOUT 超时
  - SSH 心跳
status: to-review
---

SSH 连接自动中断，通常由**会话超时**、**网络不稳**或**防火墙策略**等原因造成。要解决这个问题，关键是调整 SSH 的"保活"机制。

相关笔记：[[SSH-sshd_config认证安全配置]]

### 核心问题：SSH"保活"机制未启用

SSH 连接空闲时，为防止资源浪费，服务器或防火墙会主动切断连接。解决的核心是配置一种"心跳"机制，让客户端和服务器定期互相发送小数据包，以维持连接活跃。这可以通过修改**服务器端** (`/etc/ssh/sshd_config`) 或**客户端** (`~/.ssh/config`) 的配置来实现。

### 方案一：修改服务器端配置（推荐）

这是最根本的解决方法，可以影响所有连接此服务器的用户。你需要编辑服务器上的 `/etc/ssh/sshd_config` 文件。

主要关注以下几个参数：

| 参数 | 作用 | 建议值及说明 |
| :--- | :--- | :--- |
| **`ClientAliveInterval`** | 服务器向客户端发送"心跳"信号的时间间隔（秒）。 | 设为 `60` 或 `120`，表示每60或120秒发送一次信号。 |
| **`ClientAliveCountMax`** | 在判定连接中断前，允许客户端连续无响应的最大次数。 | 设为 `3`。这样，即使网络偶尔抖动，也有3次重试机会。 |
| **`TCPKeepAlive`** | 一个更底层的保活设置。 | 确保其值为 `yes`。 |

**配置示例**：在 `sshd_config` 文件中添加或修改以下行：
```
ClientAliveInterval 60
ClientAliveCountMax 3
TCPKeepAlive yes
```
**重要**：修改后，必须重启 SSH 服务才能生效：
```bash
sudo systemctl restart sshd
# 或者
sudo service sshd restart
```

### 方案二：修改客户端配置

如果你没有服务器管理权限，可以修改自己电脑上的客户端配置，效果只针对你个人。编辑客户端文件 `~/.ssh/config`。

相关参数与服务器端类似，但前缀不同：

| 参数 | 作用 | 建议值 |
| :--- | :--- | :--- |
| **`ServerAliveInterval`** | 客户端向服务器发送"心跳"信号的时间间隔（秒）。 | 设为 `60`。 |
| **`ServerAliveCountMax`** | 在判定连接中断前，允许服务器连续无响应的最大次数。 | 设为 `3`。 |

**配置示例**：在 `~/.ssh/config` 文件中添加以下内容，`Host *` 表示对所有连接生效：
```
Host *
    ServerAliveInterval 60
    ServerAliveCountMax 3
```

### 其他可能原因与排查

如果调整"保活"参数后问题依旧，还需排查以下几点：

*   **网络与防火墙**：检查客户端到服务器的网络是否稳定。同时，检查服务器和中间网络设备（如公司防火墙、NAT 网关）是否有针对空闲连接的超时策略。
*   **系统资源紧张**：服务器 CPU、内存不足时，可能会杀死 SSH 进程来释放资源。可以用 `top` 或 `htop` 命令检查服务器负载。
*   **内核网络参数**：某些云服务器或特定内核参数（如 `net.ipv4.tcp_tw_recycle`）可能导致 NAT 环境下的连接异常断开。除非必要，不建议修改。

### TMOUT 与 ClientAliveInterval 的区别

两者**没有直接关联**，是**完全独立**的两层超时机制，但会**叠加生效**——**谁先达到条件，谁就断开连接**。

| 特性 | **TMOUT** | **ClientAliveInterval** |
| :--- | :--- | :--- |
| **作用层级** | Shell（应用层） | SSH 协议（传输层） |
| **触发条件** | 键盘/终端**无输入** | 客户端**无网络响应**（收不到心跳包） |
| **计数方式** | 累计空闲秒数 | 间隔秒数 × 失败次数 |
| **是否可被绕过** | 是（执行 `unset TMOUT`、运行 `top` 等持续输出命令） | 否（由 sshd 强制控制，无法用户级绕过） |
| **断开表现** | `logout` / `Connection closed`（正常退出） | `Connection reset` / `timeout`（异常重置） |

### 实战中的"叠加"效应

假设配置如下：
- `TMOUT=600`（10分钟无输入断开）
- `ClientAliveInterval=300`（5分钟发心跳）+ `ClientAliveCountMax=2`（累计10分钟无响应断开）

**场景 1**：终端**空闲且网络正常**。第 10 分钟时，**`TMOUT` 先触发**，Shell 退出，连接关闭。

**场景 2**：运行 `yum update`（持续输出日志），**终端不空闲**，但中途 Wi-Fi 断了。`TMOUT` 不会触发（因为有输出），但服务端 10 分钟收不到心跳后，**`ClientAliveInterval` 触发**，强制断开。

**场景 3**：运行 `sleep 3600`（终端无输入但有活跃子进程）。`TMOUT` **依然会计时并断开**（很多 Shell 只看终端空闲，不计子进程）。此时若想挂机，需提前 `unset TMOUT`。

### 如何判断刚才是谁"杀掉"了会话？

- 如果断线前看到 **`timed out waiting for input: auto-logout`**，**铁定是 `TMOUT` 干的**（这是 Shell 打印的专属提示）。
- 如果是 `ClientAliveInterval`，通常没有任何提示，直接显示 `Connection closed by remote host` 或 `packet_write_wait`。

### 总结与建议

**优先尝试方案一，修改服务器端的配置。**

1.  **操作步骤**：登录服务器，编辑 `/etc/ssh/sshd_config`，设置 `ClientAliveInterval 60` 和 `ClientAliveCountMax 3`，然后重启 SSH 服务。
2.  **观察效果**：重新连接，看是否还会中断。
3.  **进阶排查**：如果问题依旧，再按照"其他可能原因"逐项排查网络、防火墙和服务器资源。
