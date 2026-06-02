---
tags:
  - topic/Linux
  - topic/shell
  - topic/文本处理
created: 2026-06-02
status: reviewed
---

## 核心概念

`comm` — **compare two sorted files line by line**，逐行比较两个已排序文件，是 Linux 下高效的文件比较命令。

## 输出格式

`comm` 将输出组织为三列（制表符分隔）：

| 列 | 含义 |
|----|------|
| 第1列 | 只在 **文件1** 中出现的行 |
| 第2列 | 只在 **文件2** 中出现的行 |
| 第3列 | **两个文件共有**的行（交集） |

默认情况下三列全部输出，可通过选项控制隐藏某列。

```bash
# 假设：
# a.txt:  1  2  3
# b.txt:  2  3  4

comm a_sorted.txt b_sorted.txt
# 输出：
1                          ← 第1列（只有 a 有）
                2          ← 第3列（共有）
                3          ← 第3列（共有）
        4                  ← 第2列（只有 b 有）
```

> [!tip] 视觉理解
> `comm` 的列输出用制表符对齐，但肉眼不易分辨。实践中很少用默认输出，而是配合 `-1` `-2` `-3` 选项使用。

## 常用选项

| 选项 | 含义 | 典型场景 |
|------|------|---------|
| `-1` | 不显示第1列（隐藏文件1独有的行） | 取并集等 |
| `-2` | 不显示第2列（隐藏文件2独有的行） | 取并集等 |
| `-3` | 不显示第3列（隐藏两文件共有的行） | **求对称差** |
| `-12` | 只显示第3列 | **求交集** |
| `-23` | 只显示第1列 | 求**差集** A−B |
| `-13` | 只显示第2列 | 求**差集** B−A |

## 核心使用模式

### 对称差（Symmetric Difference）— 只在一方出现

```bash
comm -3 a_sorted.txt b_sorted.txt
# 等效于：(A-B) ∪ (B-A)
```

### 交集（Intersection）— 两方都出现

```bash
comm -12 a_sorted.txt b_sorted.txt
# 等效于：A ∩ B
```

### 差集（Difference）— A 有但 B 没有

```bash
comm -23 a_sorted.txt b_sorted.txt
# 等效于：A - B
```

## 实战技巧

### 进程替换 — 避免临时文件

```bash
# 不生成临时文件，直接在内存中排序并比较
comm -3 <(sort a.txt) <(sort b.txt)
```

### 与管道的配合

```bash
# 对差集结果再次处理
comm -23 <(sort file1) <(sort file2) | wc -l   # 统计差集行数
comm -3 <(sort file1) <(sort file2) | tr '\t' '\n' | grep -v '^$'  # 清理制表符输出
```

> [!warning] 特别注意
> `comm -3` 输出中，第1列和第2列用制表符分隔。当某行属于第2列时，输出会先有一个制表符占位第1列。上述 `tr` 命令可以将制表符转成换行后过滤空行，得到干净的唯一ID列表。

### 统计两个文件的差异情况

```bash
a_only=$(comm -23 <(sort a.txt) <(sort b.txt) | wc -l)
b_only=$(comm -13 <(sort a.txt) <(sort b.txt) | wc -l)
common=$(comm -12 <(sort a.txt) <(sort b.txt) | wc -l)
echo "a独有: $a_only, b独有: $b_only, 共有: $common"
```

## 前置条件：排序

`comm` 要求输入文件**必须已排序**。未排序会导致结果完全错误。

```bash
# ✅ 正确
sort a.txt -o a_sorted.txt
sort b.txt -o b_sorted.txt
comm -3 a_sorted.txt b_sorted.txt

# ❌ 错误 — 未排序，结果不可靠
comm -3 a.txt b.txt
```

排序规则取决于 `locale` 设置，建议使用 `LC_ALL=C sort` 确保一致性。

## 与 grep 的对比

| 维度 | `comm` | `grep -Fxv -f` |
|------|--------|-----------------|
| 前提 | 要求输入已排序 | 无需排序 |
| 性能 | **O(n+m)**，适合万行以上 | **O(n×m)**，大文件慢 |
| 功能 | 同时产出差集、交集、对称差 | 单次只能算一个方向 |
| 用法 | 需配合 `sort`，略繁琐 | 直接可用，小巧灵活 |

> 性能差异的原因：`comm` 利用有序性一次遍历即可完成比较（类似归并排序的 merge 阶段）；`grep -f` 对 a.txt 的每一行都要在 b.txt 中逐行匹配。

## 相关笔记

- [[Linux文本处理三剑客]] — grep / sed / awk 与 comm 的组合使用
- [[集合运算-对称差]] — comm -3 背后的数学概念
- [[柠檬微趣-笔试-Q1-文本文件去重]] — 实际面试中的应用场景

## 参考链接

- [Linux comm 命令文档](https://man7.org/linux/man-pages/man1/comm.1.html)
