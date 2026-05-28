---
name: update-quartz-blog
description: 更新 Quartz 博客网站，包括复制本地文档到 content 目录并同步到 GitHub，同时更新 bagu 项目的 git 提交
---

# 更新 Quartz 博客

自动将本地笔记文档复制到 Quartz 博客的 content 目录，并同步到 GitHub。同时更新 bagu 项目目录的 git 提交。

## 工作流程

1. **复制文档到 content 目录**
   - 源目录：`f:\GitFiles\studyByMyself\bagu\`
   - 目标目录：`f:\GitFiles\studyByMyself\quartz\content\`
   - 需要复制的文件和目录：
     - `index.md`
     - `00-Inbox/`
     - `10-Topics/`
     - `20-Questions/`
     - `templates/`
   - 不复制的目录：
     - `90-daily/`
2. **更新 bagu 项目的 git 提交**
   - 进入 bagu 目录
   - 添加所有更改：`git add .`
   - 提交更改：`git commit -m "Update interview questions and knowledge base"`
   - 推送到远程仓库：`git push`
3. **同步 Quartz 博客到 GitHub**
   - 执行命令：`npx quartz sync`
   - 该命令会自动提交更改并推送到 GitHub 仓库
   - 会触发 Cloudflare Pages 或 GitHub Pages 自动构建部署

## 执行步骤

使用以下命令执行更新：

```bash
# 第一步：复制文档（使用 cp 命令）
cp -r "/f/GitFiles/studyByMyself/bagu/index.md" "/f/GitFiles/studyByMyself/quartz/content/"
cp -r "/f/GitFiles/studyByMyself/bagu/00-Inbox" "/f/GitFiles/studyByMyself/quartz/content/"
cp -r "/f/GitFiles/studyByMyself/bagu/10-Topics" "/f/GitFiles/studyByMyself/quartz/content/"
cp -r "/f/GitFiles/studyByMyself/bagu/20-Questions" "/f/GitFiles/studyByMyself/quartz/content/"
cp -r "/f/GitFiles/studyByMyself/bagu/templates" "/f/GitFiles/studyByMyself/quartz/content/"

# 第二步：更新 bagu 项目的 git 提交
cd "/f/GitFiles/studyByMyself/bagu"
git add .
git commit -m "Update interview questions and knowledge base"
git push

# 第三步：同步 Quartz 博客到 GitHub
cd "/f/GitFiles/studyByMyself/quartz"
npx quartz sync
```

## 注意事项

- 复制前会先清空 content 目录中对应的文件夹
- 如果只想复制特定目录，可以只执行相应的 cp 命令
- `npx quartz sync` 会自动处理 git 提交和推送
- bagu 项目的 git 提交使用固定的提交消息："Update interview questions and knowledge base"

