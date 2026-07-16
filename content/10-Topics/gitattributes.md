---
title: Git .gitattributes 文件详解
date: 2026-06-24
tags:
  - topic/git
aliases:
  - .gitattributes
  - Git 属性配置
  - Git 文件属性
status: to-review
---

`.gitattributes` 文件是 Git 中一个强大的配置文件，让你能针对**特定路径**（如文件、文件夹或文件类型）精细控制 Git 的行为。你可以把它放在项目根目录并提交到仓库，与团队共享这些设置。

相关笔记：[[Git-忽略已追踪文件]]

### 基本语法

`.gitattributes` 文件由多行规则组成，每行格式为 `模式 属性1 属性2 ...`。它主要包含以下几部分：

*   **模式 (Pattern)**：用于匹配文件路径，规则与 `.gitignore` 类似。例如 `*.txt` 匹配所有 txt 文件，`/doc/*` 匹配根目录下 `doc` 文件夹内的文件。
*   **属性 (Attributes)**：定义 Git 如何处理匹配的文件。
    *   **设置 (`属性名`)**：将该属性设为 `true`。
    *   **取消设置 (`-属性名`)**：将该属性设为 `false`。
    *   **设定值 (`属性名=值`)**：为属性指定一个字符串值。

> 注意：规则有优先级，后匹配到的规则会覆盖之前的。

### 主要功能与应用

`.gitattributes` 主要有以下几类功能：

#### 1. 管理行尾结束符 (End-of-Line)

这是最常用的功能之一，用于解决不同操作系统（Windows 用 CRLF，Linux/macOS 用 LF）带来的行尾混乱问题。

*   **`text`**：自动处理行尾。Git 会在提交时将其转换为 LF，检出时根据操作系统转换为对应格式。
*   **`text=auto`**：Git 自动检测是否为文本文件并处理行尾，是推荐的方式。
*   **`-text`**：将文件视为二进制，不进行任何行尾转换。
*   **`eol=lf`** 或 **`eol=crlf`**：强制指定该文件的换行符为 LF 或 CRLF。

```gitattributes
# 所有文本文件自动处理行尾
* text=auto

# 强制所有 .sh 文件使用 LF
*.sh text eol=lf

# 所有 .bat 文件使用 CRLF
*.bat text eol=crlf

# 将 .png 文件视为二进制，不处理行尾
*.png -text
```

#### 2. 标识二进制文件

如果 Git 将二进制文件误判为文本，可能导致仓库膨胀或合并混乱。可以用 `binary` 属性明确标识。`binary` 是 `-text -diff` 的简写，即不处理行尾也不生成差异对比。

```gitattributes
*.png binary
*.zip binary
*.jar binary
```

#### 3. 自定义文件差异对比 (Diff)

可以让 Git 对非文本文件（如图片、Word 文档）进行有意义的差异对比。

*   **配置文本转换工具**：首先，你需要一个能将二进制文件转为文本的工具（如 `docx2txt`）。
*   **设置 `diff` 属性**：在 `.gitattributes` 中为特定文件类型指定一个"过滤器"名称。
*   **配置 Git**：使用 `git config` 命令将这个"过滤器"名称与你的转换工具关联起来。

```gitattributes
# 为 .docx 文件指定 diff 过滤器 "word"
*.docx diff=word
```
```bash
# 然后配置 Git，让 "word" 过滤器使用 docx2txt 程序
git config diff.word.textconv docx2txt
```

#### 4. 指定合并策略 (Merge Strategy)

可以为特定文件指定合并时的行为。

*   **`merge=ours`**：发生冲突时，**直接使用当前分支（ours）的版本**，放弃另一分支的改动。
*   **`merge=union`**：尝试将两个版本的改动**简单地合并**在一起，适用于一些可以自动合并的场景。
*   **`merge=binary`**：将文件视为二进制，Git 不会尝试进行基于行的文本合并，发生冲突时通常需要手动解决。
*   **`-merge`**：**禁用自动合并**，即使文件是文本，发生冲突时也必须手动解决。

```gitattributes
# 锁定文件，始终保留我们的版本
*.lock merge=ours

# 对于自动生成的配置文件，尝试自动合并
config/*.json merge=union
```

#### 5. 文件编码处理

如果仓库中存在非 UTF-8 编码的文件，可以使用 `working-tree-encoding` 属性。这能让 Git 在提交时将其转换为 UTF-8 存储，检出时再转换回原编码，确保跨平台协作时编码一致。

```gitattributes
# 保持 .xhtml 文件在工作目录为 ISO-8859-1 编码
*.xhtml text working-tree-encoding=ISO-8859-1
```

#### 6. 与 Git LFS (大文件存储) 集成

可以用 `filter=lfs` 属性，让 Git LFS 自动管理匹配的大文件。

```gitattributes
*.psd filter=lfs diff=lfs merge=lfs -text
*.mp4 filter=lfs diff=lfs merge=lfs -text
```

#### 7. 其他高级功能

*   **自定义 `filter`**：可以配置 `smudge` 和 `clean` 过滤器，在文件检出和提交时执行自定义脚本，用于代码格式化、加密解密等。
*   **语言统计**：一些代码托管平台（如 GitLab）会根据 `.gitattributes` 的配置来统计项目的语言组成。
*   **语法高亮**：可以为特定文件指定编程语言，以便在网页上正确显示语法高亮。

### 文件位置与优先级

Git 会按以下优先级（从高到低）查找并应用属性设置：

1.  **`$GIT_DIR/info/attributes`**：仅对当前仓库生效，**不会**被提交。
2.  **项目根目录的 `.gitattributes`**：会被提交，与团队共享。
3.  **父目录的 `.gitattributes`**：从当前目录向上查找。
4.  **全局配置文件**：由 `core.attributesFile` 指定，对当前用户的所有仓库生效。

`.gitattributes` 是一个非常灵活的工具，合理配置它能有效避免跨平台协作时的常见问题，并优化 Git 对各类文件的管理。
