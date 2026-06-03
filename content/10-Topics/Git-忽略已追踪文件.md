---
title: Git 忽略已追踪文件
type: basic-note
date: 2025-05-07
tags: git, gitignore
---

# Git 忽略已追踪文件

## 问题

已经参与过 Git 版本管理的文件夹，即使将其加入 `.gitignore` 文件，仍不能停止 Git 追踪。

## 解决步骤

1. 确保已将文件（夹）正确添加到 `.gitignore` 文件
2. 从 Git 索引中移除文件（夹）：

   ```shell
   # 单个文件
   git rm --cached -- filePath

   # 文件夹
   git rm --cached -r -- dirPath
   ```

3. 提交此次移除操作：

   ```shell
   git commit -m "Remove filename from repository tracking"
   ```

4. 推送至远程仓库，通知其他人拉取：

   ```shell
   git push
   ```