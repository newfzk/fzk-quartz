---
title: Linux-磁盘分区与设备命名
date: 2026-06-02
updated: 2026-06-02 00:00:00
tags:
  - topic/Linux
  - topic/Linux/磁盘
status: to-review
---

## 核心概念

Linux 设备文件位于 `/dev/` 目录，遵循"设备类型 + 驱动字母 + 序号"的命名规则。

## 设备命名规则

```
/dev/sd[a-z][1-...]
         │    └─ 分区号（1=第一个分区）
         └─ 设备序号（a=第一块，b=第二块）
```

| 设备名 | 含义 |
|--------|------|
| `/dev/sda` | 第一块 SCSI/SATA/SAS 磁盘 |
| `/dev/sda1` | 第一块磁盘的第一个分区 |
| `/dev/sdb` | 第二块磁盘 |
| `/dev/sdb1` | 第二块磁盘的第一个分区 |

## 设备命名前缀

| 前缀 | 设备类型 |
|------|----------|
| `sd` | SCSI / SATA / SAS / USB 磁盘 |
| `hd` | IDE 磁盘（老旧） |
| `vd` | KVM virtio 虚拟磁盘 |
| `nvme` | NVMe SSD（如 `/dev/nvme0n1`） |
| `md` | 软件 RAID 设备 |
| `dm` | Device Mapper（LVM、LUKS） |
| `loop` | 回环设备（ISO 镜像挂载） |

## 分区编号含义

- 1-4：主分区或扩展分区（MBR 最多 4 个主分区）
- 5+：逻辑分区（在扩展分区内）

## 查看设备信息

```bash
lsblk            # 树形展示块设备
fdisk -l         # 列出所有磁盘分区表
blkid            # 显示设备 UUID 和文件系统类型
ls -l /dev/disk/by-uuid/  # 按 UUID 查看设备
```

## 面试要点

- 现代 Linux 中 `sd` 前缀不仅限于 SCSI，也包括 SATA、SAS 和 USB 磁盘
- 使用 UUID 而非设备名（如 `/dev/sda1`）挂载更可靠，因为设备名在重启后可能变化
- `/dev/sda` 是整块磁盘，`/dev/sda1` 是分区，操作分区表时使用整块磁盘

## 参考链接

- [[柠檬微趣-笔试-Q2-mount命令输出解释|柠檬微趣-笔试-Q2-mount命令输出解释]]