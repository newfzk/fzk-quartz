---
title: apk数据库与文件系统是两套真相
date: 2026-09-22
updated: 2026-09-22
aliases:
  - apk db installed
  - apk audit 盲区
  - 包数据库
related:
  - "[[案例-K8s容器时区异常-PVC挂载遮蔽usr-share]]"
  - "[[K8s卷挂载是覆盖而非合并]]"
  - "[[Alpine镜像不自带tzdata]]"
tags:
  - topic/Linux
  - topic/Docker
  - topic/故障排查
status: to-review
---

# apk 数据库与文件系统是两套真相

## 核心认知

**包数据库（`/lib/apk/db/installed`）记录的是"安装动作发生过"，不是"文件现在可见"。**

两者可以同时成立且互相矛盾：

| 事实 | 含义 |
|---|---|
| apk db 中有 `P:tzdata` 完整记录（含 `R:`/`Z:` 校验和） | 包**曾经被成功安装** |
| 文件在磁盘上不存在 | 文件**现在不可见** |

这种组合只有两类解释：

1. 文件在安装之后被**绕过 apk 的操作移除**（`rm -rf`、镜像瘦身工具）
2. 文件被**挂载遮蔽**（volumeMounts 覆盖，见 [[K8s卷挂载是覆盖而非合并]]）

> [!tip] 一条判别规则
> **`apk del tzdata` 会让 db 记录一并消失。**
> 若 db 记录完好而文件不见 → **不是 `apk del`**，必然是被"绕过 apk"的操作移走或遮蔽了。
> 这条规则能把排查范围从"是否安装过"直接推进到"安装之后发生了什么"。

## 查证命令

```bash
# apk db 里有没有这个包
grep -A5 '^P:tzdata' /lib/apk/db/installed || echo "未安装"

# 同批安装的其它包是否健在（判断是"单包丢失"还是"整批被清"）
for p in coreutils openssl gnupg binutils musl-locales ca-certificates; do
  apk info -e "$p" >/dev/null 2>&1 && echo "$p ✅" || echo "$p ❌"
done
```

字段含义：

| 字段 | 含义 |
|---|---|
| `P:` | 包名 |
| `V:` | 版本 |
| `S:` | 压缩后大小 |
| `I:` | 安装后大小（字节） |
| `R:` | 文件路径条目 |
| `Z:` | 文件校验和 |

## ⚠️ `apk audit` 的沉默陷阱

```console
$ apk audit
U etc/hosts
D etc/secfixes.d/
A etc/os-release
U etc/shadow
...
A etc/localtime
```

输出里**全是 `etc/...`，一条 `usr/share/...` 都没有**——即使 `/usr/share` 已整个消失。

> [!danger] apk-tools 2.x 的 `apk audit` 默认只审计 `/etc`
> 它**不会检查** `/usr/share`、`/usr/bin`、`/usr/lib` 等目录。
>
> **这条命令会给出"系统正常"的假象**，实际上对 `/usr` 下的损失完全沉默。排查文件缺失时**不能依赖它下结论**。

### 输出符号含义

| 符号 | 含义 |
|---|---|
| `A` | Added（新增） |
| `D` | Deleted（删除） |
| `U` | Updated（修改） |

## 如何真正验证 `/usr` 下的文件完整性

`apk audit` 覆盖不到的地方，需要换手段：

```bash
# ① 直接对比 db 中记录的文件路径与实际存在性
awk '/^P:tzdata/,/^$/' /lib/apk/db/installed | grep '^R:' | cut -d: -f2 | \
  while read f; do [ -e "/$f" ] || echo "缺失: /$f"; done

# ② 用 apk fix 重建（确认是"删除"而非"遮蔽"时才有效）
apk fix tzdata

# ③ 判断是否被遮蔽（优先级最高）
mount | grep /usr
```

> [!important] 正确顺序
> **先排除挂载遮蔽，再用 `apk fix` 重建。**
> 若问题是遮蔽，`apk fix` 会"认为文件都在"而什么都不做，白费功夫；反之若真是删除，检查挂载则毫无收获。先查挂载只需一条命令，成本最低。

## `apk add` 的连带影响

包数据库认为"已满足"时，**`apk add tzdata` 会跳过安装**：

```console
$ apk add tzdata
(1/1) Installing tzdata ...   # 或直接 "OK: 已安装"
```

即使文件实际不可见，apk 依据 db 记录判断"已安装"而不做任何事。**这进一步掩盖了问题**——运维按常规手段修复却看不出效果。

## 参考链接

- [[案例-K8s容器时区异常-PVC挂载遮蔽usr-share]] — 本矛盾的完整案例
- [[K8s卷挂载是覆盖而非合并]] — "文件不可见"的主要成因
- [[Alpine镜像不自带tzdata]] — "文件从未安装"的另一成因
