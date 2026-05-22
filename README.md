# Quartz v4

这是一个 **Quartz 4.x** 版本的静态站点生成器，专门用于将你的数字花园（Digital Garden）和笔记发布为网站。

### 📦 项目结构

| 目录/文件                | 说明                                  |
| :------------------- | :---------------------------------- |
| `quartz.config.ts`   | 主配置文件（主题、插件、站点信息）                   |
| `content/`           | Markdown 内容目录（你的笔记/文章）              |
| `quartz/components/` | React 组件（页面布局、搜索、图表等）               |
| `quartz/plugins/`    | 插件系统（transformers、filters、emitters） |
| `quartz/i18n/`       | 国际化支持（包含中文 zh-CN）                   |
| `public/`            | 构建输出目录                              |

### 🚀 本地运行命令

```bash
# 安装依赖（如果还没安装）
npm install

# 构建并启动本地预览服务器
npx quartz build --serve

# 带监听模式（文件变化自动重新构建）
npx quartz build --serve --watch

# 指定端口运行
npx quartz build --serve --port 3000
```

### ⚙️ 端口配置

端口配置在 [args.js:84-93](file:///f:/GitFiles/studyByMyself/quartz/quartz/cli/args.js#L84-L93) 中定义：

| 参数         | 默认值      | 说明                    |
| :--------- | :------- | :-------------------- |
| `--port`   | **8080** | HTTP 服务器端口            |
| `--wsPort` | **3001** | WebSocket 端口（用于热重载通知） |

**使用示例：**

```bash
# 在 3000 端口运行
npx quartz build --serve --port 3000

# 同时修改 WebSocket 端口
npx quartz build --serve --port 3000 --wsPort 3002
```

### 📝 其他常用命令

```bash
# 创建新的 Quartz 项目
npx quartz create

# 更新 Quartz 到最新版本
npx quartz update

# 同步内容到 Git 仓库
npx quartz sync
```

### 🔧 主配置文件

核心配置在 [quartz.config.ts](file:///f:/GitFiles/studyByMyself/quartz/quartz.config.ts)，你可以修改：

- `pageTitle` - 网站标题
- `baseUrl` - 网站基础 URL
- `theme` - 主题颜色和字体
- `plugins` - 启用的插件（LaTeX、目录、搜索等）

