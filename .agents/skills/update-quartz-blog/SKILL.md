---
name: update-quartz-blog
description: 更新 Quartz 博客网站，包括自动同步 index.md 链接、复制文档到 content 目录、更新 git 提交
---

# 更新 Quartz 博客

自动扫描 10-Topics/ 和 20-Questions/ 目录的文件变更，更新 index.md 中的链接列表，然后将笔记文档复制到 Quartz 博客的 content 目录，并同步到 GitHub。同时更新 bagu 项目目录的 git 提交。

**禁止**将 30-Secret-Questions/ 目录下的文件复制到 Quartz 博客的 content 目录下；

## 执行步骤

使用以下命令执行更新：

```bash
# 第一步：根据 10-Topics/ 和 20-Questions/ 的文件变更，更新 index.md
node "/f/GitFiles/studyByMyself/bagu/scripts/update-index.js"

# 第二步：复制文档（使用 cp 命令）
cp -r "/f/GitFiles/studyByMyself/bagu/index.md" "/f/GitFiles/studyByMyself/quartz/content/"
cp -r "/f/GitFiles/studyByMyself/bagu/00-Inbox" "/f/GitFiles/studyByMyself/quartz/content/"
cp -r "/f/GitFiles/studyByMyself/bagu/10-Topics" "/f/GitFiles/studyByMyself/quartz/content/"
cp -r "/f/GitFiles/studyByMyself/bagu/20-Questions" "/f/GitFiles/studyByMyself/quartz/content/"
cp -r "/f/GitFiles/studyByMyself/bagu/templates" "/f/GitFiles/studyByMyself/quartz/content/"

# 第三步：更新 bagu 项目的 git 提交
cd "/f/GitFiles/studyByMyself/bagu"
git add .
git commit -m "Update interview questions and knowledge base"
# git push

# 第四步：同步 Quartz 博客到 GitHub
cd "/f/GitFiles/studyByMyself/quartz"
npx quartz sync
```

## 注意事项

- 复制前会先清空 content 目录中对应的文件夹
- 如果只想复制特定目录，可以只执行相应的 cp 命令
- 第一步会自动添加新笔记链接到 index.md，移除已删除笔记的链接，并保留已有的自定义显示名
- 新笔记的显示名从 frontmatter 的 `title` 字段自动提取，若无则使用文件名
- `npx quartz sync` 会自动处理 git 提交和推送
- bagu 项目的 git 提交使用固定的提交消息："Update interview questions and knowledge base"

