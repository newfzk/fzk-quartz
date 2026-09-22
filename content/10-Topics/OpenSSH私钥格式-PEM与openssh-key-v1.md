---
title: OpenSSH私钥格式-PEM与openssh-key-v1
date: 2026-08-04
updated: 2026-09-22
aliases:
  - openssh-key-v1
  - PEM 私钥格式
  - PKCS#1 PKCS#8
related:
  - "[[JGit内置JSch不支持OpenSSH新格式私钥]]"
  - "[[SSH-sshd_config认证安全配置]]"
tags:
  - topic/SSH
  - topic/密钥管理
status: to-review
---

# OpenSSH 私钥格式：PEM 与 openssh-key-v1

## 用首行标识区分格式

私钥文件的第一行就决定了它属于哪种格式，也是判断客户端能否解析的第一依据：

| 首行标识 | 格式名 | 编码标准 | 兼容性 |
|---|---|---|---|
| `-----BEGIN OPENSSH PRIVATE KEY-----` | OpenSSH 新格式 | `openssh-key-v1` | 新，但老库（如 JSch 0.1.5x）**不支持** |
| `-----BEGIN RSA PRIVATE KEY-----` | 传统 PEM | PKCS#1 | 广，绝大多数老库支持 |
| `-----BEGIN PRIVATE KEY-----` | PKCS#8 | PKCS#8 | 部分库（含 JSch 0.1.5x）**不支持** |
| `-----BEGIN EC PRIVATE KEY-----` | 传统 PEM | SEC1 | 广 |
| `-----BEGIN DSA PRIVATE KEY-----` | 传统 PEM | 传统 DSA | 广（已淘汰） |

## 为什么会有新格式

OpenSSH 7.8 起，`ssh-keygen` **默认输出 `openssh-key-v1` 新格式**，而不是传统 PEM。新格式的设计动因：

1. 统一承载多种密钥类型（RSA / ECDSA / **ed25519**），传统 PEM 无法表示 ed25519
2. 原生支持 bcrypt 加密私钥（`-a` 参数控制迭代轮数），抵抗离线爆破
3. 自带完整性校验，防止私钥被篡改

> [!warning] 兼容性陷阱
> 新格式虽好，但**大量 Java/Go 生态的老 SSH 库尚未跟进**。这就是"命令行 ssh 能用、程序里报 `invalid privatekey`"这类问题的根源所在。

## 生成命令对照

```sh
# 默认：OpenSSH 新格式（ed25519，bcrypt 加密）
ssh-keygen -t ed25519 -C "comment"

# 传统 PEM 格式 RSA（兼容老库，如 JGit 内置 JSch）
ssh-keygen -t rsa -b 4096 -m PEM -C "comment"

# 只转换格式、不重新生成密钥（openssh-key-v1 → PEM）
ssh-keygen -p -m PEM -f ~/.ssh/id_rsa
```

## 如何选择

- **纯命令行 / OpenSSH 生态**：用默认新格式 + ed25519，安全性最好
- **需要被 Java 库消费（JGit、JSch、部分 Gradle 插件）**：用 `-m PEM` 生成 RSA
- **已有新格式密钥但消费方不支持**：优先 `ssh-keygen -p -m PEM` 转换；但 ed25519 无法转换为 PEM，必须重新生成 RSA

## 参考链接

- [[JGit内置JSch不支持OpenSSH新格式私钥]] — 该格式差异导致的真实故障案例
