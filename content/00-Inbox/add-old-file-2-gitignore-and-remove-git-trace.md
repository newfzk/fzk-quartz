---
title: 向gitignore中添加已有文件并移除git追踪
type: basic-note
date: 2025-05-07
tags: git, gitignore
---

# 向gitignore中添加已有文件并移除git追踪

## 问题 已经参与过git版本管理的文件夹 哪怕后面将其加入gitignore文件 仍不能停止git追踪

1. 确保已经将文件（夹）正确添加入`.gitignore`文件
2. 从git索引中移除文件（夹）`git rm --cached [-r 如果是文件夹] -- filePathOrDirPath`
3. 提交此次移除操作`git commit -m "Remove filename from repository tracking"`
4. `git push`同步至远程仓库 通知其他人拉取
