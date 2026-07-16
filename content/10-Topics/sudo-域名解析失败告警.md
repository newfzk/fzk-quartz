---
title: sudo 域名解析失败告警排查
date: 2026-06-24
tags:
  - topic/Linux
  - topic/故障排查
aliases:
  - sudo unable to resolve host
  - sudo 主机名解析
  - sudo 告警 Connection timed out
status: to-review
---

使用 `sudo` 后，出现域名解析失败的告警，日志如下：

```sh
root@cloud-test03:~# sudo ls
sudo: unable to resolve host cloud-test03: Connection timed out
```

### 原因

1. **`sudo` 需要解析主机名**
   `sudo` 在执行时会记录日志（通常通过 syslog），日志条目中需要包含当前主机的主机名。因此它会调用 `gethostname()` 获取主机名（这里是 `cloud-test03`），然后尝试通过 DNS 或 `/etc/hosts` 将其解析为 IP。

2. **解析顺序问题**
   Linux 系统根据 `/etc/nsswitch.conf` 中 `hosts` 行的配置决定解析顺序（例如 `hosts: files dns` 表示先查 `/etc/hosts`，再查 DNS）。

   - 如果 `/etc/hosts` 中 **没有** `cloud-test03` 的条目，系统就会转向 DNS 查询。
   - 机器当前 DNS 服务器可能不可达、响应极慢或配置错误，导致查询过程等待直到"Connection timed out"。

3. **为什么是"Connection timed out"而不是"Unknown host"**
   因为 DNS 请求发出后，目标 DNS 服务器没有响应（防火墙丢弃、服务器宕机、网络中断等），TCP/UDP 连接或请求超时。

### 常见原因场景

- 手动修改过主机名（如 `hostnamectl set-hostname cloud-test03`），但忘了同步更新 `/etc/hosts`。
- 系统启动时 DHCP 或网络配置没有正确更新 `/etc/hosts`。
- DNS 服务器配置错误（`/etc/resolv.conf` 里的 nameserver 不可达）。
- 网络故障（如网卡未启动、路由不通）。

### 解决方案

在 `/etc/hosts` 中添加本机主机名解析：

```bash
echo "127.0.0.1 cloud-test03" >> /etc/hosts
# 或者使用内网 IP
echo "<内网IP> cloud-test03" >> /etc/hosts
```

### 推荐做法

修改主机名后，养成同步更新 `/etc/hosts` 的习惯，确保 `hosts` 文件中包含 `127.0.0.1 <hostname>` 或 `<实际IP> <hostname>` 的条目。
