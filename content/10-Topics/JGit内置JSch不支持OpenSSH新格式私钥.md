---
title: JGit内置JSch不支持OpenSSH新格式私钥
date: 2026-08-04
updated: 2026-09-22
aliases:
  - JGit invalid privatekey
  - JGit 私钥格式限制
  - JSch 0.1.5x 私钥支持
related:
  - "[[OpenSSH私钥格式-PEM与openssh-key-v1]]"
  - "[[SSH-sshd_config认证安全配置]]"
tags:
  - topic/Git
  - topic/SSH
  - topic/java
  - topic/故障排查
status: to-review
---

# JGit 内置 JSch 不支持 OpenSSH 新格式私钥

## 核心结论

用 JGit（`org.eclipse.jgit.ssh.jsch`）做 Git 操作时报 `invalid privatekey`，而命令行 `ssh` 手动 clone 却成功——**根因是两者使用的 SSH 实现不同**：

| 使用方 | SSH 实现 | 能否解析 OpenSSH 新格式 / ed25519 |
|---|---|---|
| 命令行 `ssh` | OpenSSH 库 | ✅ 支持 |
| JGit（`jgit.ssh.jsch`） | JSch 0.1.5x（内置） | ❌ 不支持 |

## JSch 0.1.5x 支持的格式

只支持**传统 PEM 格式**：

- `-----BEGIN RSA PRIVATE KEY-----`（PKCS#1）
- `-----BEGIN EC PRIVATE KEY-----`
- `-----BEGIN DSA PRIVATE KEY-----`

## JSch 0.1.5x 不支持的格式

- `-----BEGIN OPENSSH PRIVATE KEY-----`（openssh-key-v1 新格式）——**OpenSSH 7.8+ 的 `ssh-keygen` 默认产出**
- `-----BEGIN PRIVATE KEY-----`（PKCS#8）
- **ed25519 密钥**（无论什么格式）

## 解决：显式生成 PEM 格式的 RSA 密钥

```sh
ssh-keygen -t rsa -b 4096 -m PEM -C "fzk-gitlab-10202212-rsa"
```

关键参数：

- `-t rsa`：避开 ed25519，因为 JSch 完全不支持 ed25519
- `-b 4096`：密钥长度
- `-m PEM`：**输出传统 PEM 格式而非 openssh-key-v1**，这是解决 `invalid privatekey` 的决定性参数

## 排查要点

1. **先看私钥文件首行**是 `BEGIN OPENSSH PRIVATE KEY` 还是 `BEGIN RSA PRIVATE KEY`，一行命令即可定位格式问题。
2. **不要因为命令行 ssh 成功就排除密钥问题**——命令行与 JGit 走的是两套解析实现，这是最容易误导排查方向的地方。
3. 若必须使用 ed25519 或新格式私钥，需替换 JGit 的 SSH 会话工厂（如改用 Apache MINA sshd 或系统 ssh 代理），而非改密钥格式。

## 参考链接

- [[OpenSSH私钥格式-PEM与openssh-key-v1]] — 各类密钥格式的详细对比
