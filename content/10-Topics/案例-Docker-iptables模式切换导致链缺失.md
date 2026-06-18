---
title: 案例：Docker iptables 模式切换导致链缺失
tags:
  - topic/Docker
  - topic/计算机网络
  - topic/故障排查
status: to-review
---

# 案例：Docker iptables 模式切换导致链缺失

## 问题分析

从错误日志来看，根本原因确实与 **iptables 模式变更**直接相关。

### 关键线索

你的 `kube-node` 角色中有以下逻辑：

```yaml
- name: 切换 iptables 为 legacy 模式 (Ubuntu 22.04)
  alternatives:
    name: "{{ item }}"
    path: "/usr/sbin/{{ item }}-legacy"
  with_items:
    - iptables
    - ip6tables
  when: ansible_distribution_release == "jammy"
```

### 报错根因

核心错误：
```
iptables --wait -t nat -I DOCKER -i br-dfe297749658 -j RETURN: iptables: No chain/target/match by that name.
```

**问题链路如下：**
1. 目标节点是 Ubuntu 22.04 (jammy)，**默认使用 nftables** 作为 iptables 后端
2. 首次部署时，Docker 启动后通过 **nft 模式** 在 `nat` 表中创建了 `DOCKER` 链，Harbor 安装正常
3. 之后 `kube-node` 角色将 iptables 切换为 **legacy 模式**（`iptables-legacy`）
4. 重新部署 Harbor 时，Docker 使用 **legacy 模式的 iptables** 去查找 `nat` 表中的 `DOCKER` 链——**但这张链是在 nft 模式下创建的，legacy 模式下根本看不到它**
5. 于是报错 `No chain/target/match by that name`

简单说就是：**Docker 的 DOCKER 链存在在 nftables 中，但 iptables 已被切到 legacy 模式，两边看不到对方的规则。**

---

## 可行方案（按推荐优先级排序）

### 方案 1：重启 Docker 服务（最简单，优先尝试）

切换 iptables 模式后，Docker 守护进程需要在新的 iptables 后端下重新初始化网络规则。

```bash
systemctl restart docker
```

重启后 Docker 会用当前 iptables 模式重新创建所有链（包括 `DOCKER` 链），然后再执行 Harbor 安装即可。

**部署流程建议**：在 `kube-node` 切换 iptables 后添加一个重启 Docker 的步骤，再执行 Harbor 部署。

### 方案 2：增加 `docker restart` 到 Ansible 编排中

在 harbor tasks 的 `install.sh` 执行之前，增加一个确保 Docker iptables 链正确的 step：

```yaml
- name: 确保 docker 的 iptables 链与当前 iptables 模式一致
  shell: systemctl restart docker
  when: ansible_distribution_release == "jammy"

- name: 等待 docker 恢复
  shell: systemctl is-active docker
  register: docker_active
  until: docker_active.stdout == "active"
  retries: 6
  delay: 5
```

### 方案 3：检查 Docker daemon 的 iptables 配置

在目标节点上检查 Docker 的 iptables 配置：

```bash
# 查看当前 iptables 模式
iptables --version

# 查看 nat 表是否有 DOCKER 链
iptables -t nat -L -n | grep DOCKER

# 查看 docker daemon 的 iptables 配置
docker info | grep -i iptables
```

如果 `docker info` 显示 `iptables` 为关闭状态，检查 `/etc/docker/daemon.json` 是否有 `"iptables": false`。

### 方案 4：清理旧 iptables 规则后重启 Docker

如果方案 1 仍不生效，说明 nft 模式下的旧规则残留导致冲突，可以彻底清理：

```bash
# 清理所有 iptables 规则（谨慎操作）
iptables -F
iptables -t nat -F
iptables -t mangle -F
iptables -X

# 重启 Docker
systemctl restart docker
```

### 方案 5：修改 kube-node 逻辑，切换模式后确保 Docker 重新初始化

在 kube-node 的 iptables 切换步骤之后，增加 Docker 重启步骤：

```yaml
- name: 切换 iptables 为 legacy 模式后重启 docker
  systemd:
    name: docker
    state: restarted
    daemon_reload: yes
  when: ansible_distribution_release == "jammy"
```

---

## 相关笔记

- [[Docker网络模式-bridge]] — Docker 网络模式详解
- [[iptables详解]] — iptables 四表五链、包流转
- [[Linux网络数据包处理]] — MOC：Linux 网络栈与包处理
- [[Linux-update-alternatives命令详解]] — `alternatives` 命令管理多版本切换

## 总结

| 排查步骤 | 命令/操作 | 预期 |
|---------|----------|------|
| 1. 确认 iptables 模式 | `iptables --version` | 应为 `legacy` |
| 2. 检查 DOCKER 链存在 | `iptables -t nat -L -n \| grep DOCKER` | 应有该链 |
| 3. 确认问题 | 没有 DOCKER 链 → iptables 模式切换导致 | — |
| 4. 解决 | `systemctl restart docker` | 自动重建 DOCKER 链 |

**最直接的解决方案**就是在 Harbor 安装前重启一次 Docker，让 Docker 在当前 iptables 模式下重建网络规则。

## TODO

- [ ] docker 链是什么，待补充
