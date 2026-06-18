---
title: Linux update-alternatives 命令详解
date: 2026-06-11
updated: 2026-06-11
tags:
  - topic/Linux
  - topic/Linux/命令
status: to-review
---

## 概述

**`update-alternatives`** 是 Debian/Ubuntu 系统用于维护**符号链接（symlink）** 的软件管理工具，用于管理系统中同一命令的多个版本共存时，**哪个版本是默认使用的**。

> [!info] 核心原理
> 系统中可能存在多个提供相同功能的程序版本（如 Java 8 和 Java 11、iptables-legacy 和 iptables-nft）。`update-alternatives` 通过维护 `/etc/alternatives/` 目录下的符号链接，控制 `/usr/bin/` 等路径下的命令指向哪个实际版本。

---

## 一、工作原理

### 1.1 机制

```
用户执行 java
    │
    ▼
/usr/bin/java         ← 系统标准路径（PATH 中的命令）
    │         (符号链接)
    ▼
/etc/alternatives/java   ← update-alternatives 管理的链接
    │         (符号链接)
    ▼
/usr/lib/jvm/java-11-openjdk-amd64/bin/java
                         ← 实际的候选版本
```

### 1.2 三个层次

| 层次 | 路径 | 说明 |
|:----:|------|------|
| **主链接** | `/usr/bin/<name>` | 用户直接执行的命令，指向 `/etc/alternatives/<name>` |
| **管理链接** | `/etc/alternatives/<name>` | `update-alternatives` 控制的目标，指向所选候选版本的路径 |
| **候选版本** | 实际程序的安装路径 | 系统中安装的各个版本 |

---

## 二、常用命令

### 2.1 查看命令的可用版本

```bash
# 查看某个命令的备选版本列表和当前选择
update-alternatives --display iptables

# 或使用 config 进入交互式选择界面
update-alternatives --config iptables
```

### 2.2 切换版本

```bash
# 非交互方式指定版本
update-alternatives --set iptables /usr/sbin/iptables-legacy

# 交互方式（列出所有候选，让你选编号）
update-alternatives --config iptables
```

### 2.3 手动注册/移除候选版本

```bash
# 注册一个新的候选版本
#   --install <主链接> <名称> <候选路径> <优先级>
update-alternatives --install /usr/bin/java java /usr/lib/jvm/java-11-openjdk-amd64/bin/java 1100

# 移除一个候选版本
update-alternatives --remove java /usr/lib/jvm/java-8-openjdk-amd64/bin/java

# 移除整个替代组（所有候选）
update-alternatives --remove-all java
```

> [!tip] 优先级（Priority）
> 注册时指定的**优先级数字**：数字越大优先级越高。当没有手动设置时，系统自动选择优先级最高的版本。
> - `--auto <name>`：恢复为自动模式（按优先级选择）
> - `--set <name> <path>`：手动指定版本（覆盖自动模式）

### 2.4 查看状态

```bash
# 查看所有注册的替代项
update-alternatives --get-selections

# 查看特定命令的替代组
update-alternatives --display java
update-alternatives --list java        # 列出所有候选路径
update-alternatives --query java       # 机器可读的详细输出
```

---

## 三、与 iptables 的关系

### 3.1 背景：为什么 iptables 需要 alternatives

现代 Debian/Ubuntu 系统（Debian 10+/Ubuntu 20.04+）默认使用 **nftables** 作为内核防火墙框架，但仍提供一个**兼容层**，让旧版 `iptables` 命令语法可以正常工作：

| 候选版本 | 说明 |
|---------|------|
| **`iptables-legacy`** | 使用传统 **netfilter** API，内核模块为 `ip_tables` |
| **`iptables-nft`** | 使用新版 **nftables** 内核 API（通过兼容层翻译旧语法），内核模块为 `nf_tables` |

`update-alternatives` 控制着当用户输入 `iptables` 时，系统实际调用哪个版本：

```text
iptables 命令 → update-alternatives 选择 → iptables-legacy / iptables-nft
```

### 3.2 legacy vs nft 模式对比

| 对比项 | legacy 模式 | nft 模式 |
|--------|:-----------:|:--------:|
| 内核框架 | 传统 netfilter（`ip_tables`） | nftables（`nf_tables`） |
| 命令路径 | `/usr/sbin/iptables-legacy` | `/usr/sbin/iptables-nft` |
| 兼容性 | 与 Docker、K8s 等旧工具兼容性最好 | 与 nftables 工具链一致 |
| 性能 | 规则多时性能下降明显 | 使用 Set/Map，大数据量性能更好 |
| 功能集 | 传统 iptables 功能 | 支持 nftables 新特性（sets、maps、verdict maps） |

> [!important] Docker 与 legacy 模式
> Docker 的 bridge 网络模式依赖于 iptables NAT 规则来管理容器端口映射。在某些系统（特别是 Debian 12 等新版系统）上，如果默认使用 `iptables-nft`，可能会导致 Docker 的端口转发规则不生效，需要切换为 `iptables-legacy` 模式。
>
> 详见 [[Docker网络模式-bridge#iptables 与 Docker 的协作]]。

### 3.3 查看 iptables 的 alternatives — 实际输出逐行解读

执行 `update-alternatives --display iptables`，实际输出如下：

```text
root@cloud-test02:~# update-alternatives --display iptables
iptables - auto mode
  link best version is /usr/sbin/iptables-nft
  link currently points to /usr/sbin/iptables-nft
  link iptables is /usr/sbin/iptables
  slave iptables-restore is /usr/sbin/iptables-restore
  slave iptables-save is /usr/sbin/iptables-save
/usr/sbin/iptables-legacy - priority 10
  slave iptables-restore: /usr/sbin/iptables-legacy-restore
  slave iptables-save: /usr/sbin/iptables-legacy-save
/usr/sbin/iptables-nft - priority 20
  slave iptables-restore: /usr/sbin/iptables-nft-restore
  slave iptables-save: /usr/sbin/iptables-nft-save
```

#### 逐行解释

| 行内容 | 含义 |
|--------|------|
| **替代组头部** | |
| `iptables - auto mode` | 替代组名称为 `iptables`，当前为 **auto 模式**（由优先级自动决定用哪个版本）。如果曾用 `--set` 手动指定过，会显示 `manual mode` |
| **主链接状态** | |
| `link best version is /usr/sbin/iptables-nft` | 根据优先级排序，**最佳版本**是 `iptables-nft`（因为它的优先级 20 > legacy 的 10） |
| `link currently points to /usr/sbin/iptables-nft` | 当前主链接 **实际指向** `iptables-nft`。因为 auto 模式下，系统自动选择了优先级最高的版本 |
| `link iptables is /usr/sbin/iptables` | 主链接本身的路径。即 `/etc/alternatives/iptables` 最终指向的是 `/usr/sbin/iptables`，而这个文件本身又是一个指向具体候选版本的 symlink |
| **附属链接（Slave）状态** | |
| `slave iptables-restore is /usr/sbin/iptables-restore` | 附属命令 `iptables-restore` 的当前指向 |
| `slave iptables-save is /usr/sbin/iptables-save` | 附属命令 `iptables-save` 的当前指向 |
| **候选版本 1：legacy** | |
| `/usr/sbin/iptables-legacy - priority 10` | 第一个候选版本路径，优先级 **10**（较低） |
| `slave iptables-restore: /usr/sbin/iptables-legacy-restore` | 如果切换到 legacy 模式，`iptables-restore` 会跟随指向此路径 |
| `slave iptables-save: /usr/sbin/iptables-legacy-save` | 如果切换到 legacy 模式，`iptables-save` 会跟随指向此路径 |
| **候选版本 2：nft** | |
| `/usr/sbin/iptables-nft - priority 20` | 第二个候选版本路径，优先级 **20**（较高） |
| `slave iptables-restore: /usr/sbin/iptables-nft-restore` | 如果当前是 nft 模式，`iptables-restore` 指向此路径 |
| `slave iptables-save: /usr/sbin/iptables-nft-save` | 如果当前是 nft 模式，`iptables-save` 指向此路径 |

> [!tip] Master-Slave 关联机制
> `update-alternatives` 支持 **Master-Slave 关联**：当你切换主命令（master）时，附属命令（slave）会**自动跟随切换**。
>
> 即选择 `iptables` → `iptables-legacy` 时，`iptables-restore` 和 `iptables-save` **自动**指向对应的 legacy 版本，无需单独设置。
>
> 这就是为什么 `--display` 输出中每一个候选版本都列出了它的 slave 文件。

#### 从输出中能读出的信息

1. **当前使用的是 nft 模式**：因为 `currently points to` 显示为 `iptables-nft`
2. **模式是 auto**：系统自动选择的（未被手动锁定）
3. **切换到 legacy 后会连带影响哪些命令**：`iptables-restore`、`iptables-save` 会一起切换
4. **优先级对比**：nft（20）> legacy（10），所以 auto 模式下默认选 nft

### 3.4 切换 iptables 的 alternatives

```bash
# 切换到 legacy 模式
sudo update-alternatives --set iptables /usr/sbin/iptables-legacy
sudo update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy
sudo update-alternatives --set arptables /usr/sbin/arptables-legacy
sudo update-alternatives --set ebtables /usr/sbin/ebtables-legacy

# 验证
iptables --version
# iptables v1.8.7 (legacy)   ← 显示 legacy 表示已切换
# iptables v1.8.7 (nft)      ← 显示 nft 表示当前是 nftables 模式
```

---

## 四、其他常见应用场景

### 4.1 管理 Java 版本

最常见的场景：系统中安装了多个 JDK 版本。

```bash
# 注册多个 JDK 版本
sudo update-alternatives --install /usr/bin/java java /usr/lib/jvm/jdk-11/bin/java 1100
sudo update-alternatives --install /usr/bin/java java /usr/lib/jvm/jdk-17/bin/java 1700

# 同时注册 javac
sudo update-alternatives --install /usr/bin/javac javac /usr/lib/jvm/jdk-17/bin/javac 1700

# 交互式切换
sudo update-alternatives --config java
```

### 4.2 管理 Python 版本

```bash
# 注册 Python 版本
sudo update-alternatives --install /usr/bin/python python /usr/bin/python3.10 1
sudo update-alternatives --install /usr/bin/python python /usr/bin/python3.11 2

# 切换
sudo update-alternatives --config python
```

### 4.3 管理编辑器

```bash
# Debian 系统默认用 alternatives 管理默认编辑器
sudo update-alternatives --config editor
# 候选：/bin/nano, /usr/bin/vim.basic, /usr/bin/vim.tiny 等
```

### 4.4 管理 gcc/g++ 版本

```bash
# 查看当前 gcc 版本
gcc --version

# 查看所有候选版本
sudo update-alternatives --config gcc
```

---

## 五、与 iptables 相关的完整操作流程

当在 Docker 环境中遇到 iptables 相关问题时，典型排查与切换步骤：

```bash
# 1. 查看当前 iptables 版本模式
iptables --version
# iptables v1.8.7 (nft)   ← 如果 Docker 有问题，可能需要切换

# 2. 查看 alternatives 状态
update-alternatives --display iptables

# 3. 全部切换到 legacy 模式
sudo update-alternatives --set iptables /usr/sbin/iptables-legacy
sudo update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy
sudo update-alternatives --set arptables /usr/sbin/arptables-legacy
sudo update-alternatives --set ebtables /usr/sbin/ebtables-legacy

# 4. 验证切换结果
iptables --version
# iptables v1.8.7 (legacy)   ← 确认已切换

# 5. 重启 Docker 服务（让 Docker 重新加载 iptables 规则）
sudo systemctl restart docker
```

---

## 六、面试要点速记

1. **本质**：`update-alternatives` 就是符号链接管理器，通过修改 `/etc/alternatives/` 下的软链接实现版本切换
2. **iptables 的两种模式**：legacy（传统 netfilter）vs nft（nftables 兼容层），通过 alternatives 切换
3. **Docker 的依赖**：Docker bridge 网络依赖 iptables NAT，新版系统可能需要切换到 legacy 模式
4. **优先级机制**：`--install` 时指定优先级，`--auto` 自动选优先级最高的，`--set` 手动锁定
5. **不重启生效**：切换 alternatives 是立即生效的，不需要重启系统

## 参考链接

- [[iptables详解]] — iptables 防火墙命令详解
- [[iptables端口转发]] — DNAT/SNAT 端口转发配置
- [[Docker网络模式-bridge]] — Docker bridge 网络与 iptables 的关系
- [[netfilter框架详解]] — netfilter 内核框架详解
- [[Linux网络数据包处理]] — 网络包处理全景 MOC
