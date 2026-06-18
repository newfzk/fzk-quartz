---
title: SSH — sshd_config 认证安全配置
date: 2026-06-09
aliases:
  - sshd_config
  - LoginGraceTime
  - PermitRootLogin
  - StrictModes
  - SSH 安全配置
  - SSH 服务器配置
tags:
  - topic/Linux
  - topic/安全
  - topic/计算机网络
status: to-review
---

# SSH — sshd_config 认证安全配置

`sshd_config` 是 OpenSSH 服务端（`sshd`）的核心配置文件，位于 `/etc/ssh/sshd_config`。以下三个指令涉及 SSH 连接认证阶段的安全防护。

---

## LoginGraceTime 120

```ini
LoginGraceTime 120
```

**含义：** 设置 SSH 认证超时时间，单位秒。

- 客户端建立 TCP 连接后，必须在 **120 秒内** 完成身份认证
- 超时后服务器主动断开连接，释放该连接占用的资源

**安全作用：**
- 防止 **慢速攻击（Slow DoS）**——攻击者建立大量连接但故意不完成认证，耗尽服务器的 `MaxStartups` 连接槽位
- 释放因客户端异常中断而残留的半开连接
- 默认值为 120 秒，生产环境建议保持此值或更短（如 60 秒）

> [!tip] 调优建议
> 如果网络延迟较高且使用密码认证（需手动输入），可适当放宽至 120~180 秒；若仅使用密钥认证（无交互等待），可缩短至 30 秒以减少攻击窗口。

---

## PermitRootLogin prohibit-password

```ini
PermitRootLogin prohibit-password
```

**含义：** 控制 root 用户能否通过 SSH 登录。

`prohibit-password` 是 OpenSSH 7.0+ 引入的安全增强选项，表示：

| 认证方式 | 是否允许 |
|---------|:-------:|
| 公钥认证（publickey） | ✅ 允许 |
| 密码认证（password） | ❌ 拒绝 |
| 键盘交互认证（keyboard-interactive） | ❌ 拒绝 |
| 主机认证（hostbased） | ❌ 拒绝 |

**安全作用：**
- 杜绝针对 root 账号的 **密码暴力破解**（这是 SSH 攻击最常用的手段）
- 保留 root 通过密钥登录的能力，方便自动化运维和紧急管理
- 比 `PermitRootLogin yes` 安全，比 `PermitRootLogin no` 灵活

> [!important] 最佳实践
> 即使使用密钥登录 root，也建议：
> 1. 为密钥设置密码短语（passphrase）
> 2. 使用 `Match Address` 限制允许登录 root 的源 IP 范围
> 3. 推荐日常使用普通用户登录 + `sudo`，仅在必要时通过密钥登录 root

**其他可选值：**

| 值 | 行为 |
|----|------|
| `yes` | 允许 root 通过任何方式登录（不安全） |
| `no` | 禁止 root 登录（最严格，需 sudo 提权） |
| `prohibit-password` | 仅允许密钥认证（**推荐**） |
| `forced-commands-only` | 仅允许执行指定命令（用于特定场景） |

---

## StrictModes yes

```ini
StrictModes yes
```

**含义：** 在允许用户登录前，严格检查用户家目录和 SSH 相关文件的权限。

当 `StrictModes yes` 时，`sshd` 会在认证前验证以下文件的权限是否安全：

| 检查对象 | 权限要求 |
|---------|---------|
| `~/.ssh/` 目录 | 不能对 group/others 可写 |
| `~/.ssh/authorized_keys` | 不能对 group/others 可写 |
| `~/.ssh/config` | 不能对 group/others 可写 |
| 用户家目录 `~` | 不能对 group/others 可写 |
| `~/.rhosts`, `~/.shosts` | 不能对 group/others 可写 |

**安全作用：**
- 防止 **权限配置错误** 导致未授权访问——例如用户误将 `authorized_keys` 设为 `777`，攻击者可以注入自己的公钥
- 阻止低权限用户通过修改其他用户的 SSH 配置来提权

> [!warning] 常见问题
> 如果因误操作导致家目录权限变松（如 `chmod 777 ~`），即便密钥正确也无法登录，`sshd` 会拒绝连接并记录类似日志：
> ```
> Authentication refused: bad ownership or modes for directory /home/user
> ```
> 修复方式：`chmod 755 ~` 或 `chmod 700 ~`

---

## 配置验证与重载

修改 `sshd_config` 后必须执行以下步骤：

```bash
# 检查配置语法
sudo sshd -t

# 重新加载配置（不中断现有连接）
sudo systemctl reload sshd
```

> [!tip]
> 每次修改前先备份：`sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak`

---

## 关联知识点

- [[Linux-文件描述符fd详解]] — SSH 底层通过 socket 文件描述符实现连接管理
- [[iptables详解]] — 可通过 iptables 进一步限制 SSH 访问源 IP
- [[tcpdump-过滤表达式]] — 抓包分析 SSH 认证流量
- [[tcpdump-抓包保存与读取]] — SSH 暴力破解流量分析
