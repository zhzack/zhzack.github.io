# Bluegrey Blog 🚀

基于 **[Astro 5](https://astro.build/)** + **[Fuwari](https://github.com/saicaca/fuwari)** 现代化架构重构的个人博客。

## ✨ 特性与亮点

- ⚡️ **极速体验**：Astro 5 孤岛架构与零运行时 JS，静态 HTML 秒开。
- 🎨 **现代视觉**：卡片流式布局、响应式设计、平滑页面切换（Swup）、深浅色主题自适应与动态色相切换。
- 🔍 **客户端离线搜索**：集成 **Pagefind** 全文检索，毫秒级快速匹配。
- 📝 **增强 Markdown & MDX**：
  - 代码高亮：Expressive Code（行号、可折叠代码块、一键复制、语言徽章）。
  - 数学公式：KaTeX 完美渲染。
  - 目录大纲 (TOC)：自动生成文章层级目录。
- 🚀 **双重部署支持**：
  - **GitHub Pages**：推送到 `master` 分支自动触发 GitHub Actions 构建并发布。
  - **Cloudflare Pages**：天然兼容 Cloudflare 边缘计算与全球 CDN 加速。

---

## 🛠️ 本地开发与指令

```bash
# 安装依赖
pnpm install

# 启动本地热更新开发服务 (http://localhost:4321)
pnpm dev

# 创建新文章 (自动填充标准 Front-matter)
pnpm new-post <文章英文文件名>

# 生产环境静态打包与搜索索引构建 (输出至 ./dist)
pnpm build

# 预览生产构建结果
pnpm preview
```

---

## ✍️ 写新文章指南

在 `src/content/posts/` 目录下创建 `.md` 文件，头部加入标准 Frontmatter：

```markdown
---
title: 我的新文章标题
published: 2026-11-17
description: 这是文章的简短摘要
tags: [前端, Astro, 架构]
category: 技术
draft: false
---

正文内容...
```

- `draft: true`：草稿模式，在生产构建时不会被公开发布。
- `image: ""`：可指定文章封面大图路径（放在 `src/assets/images/` 或 `public/`）。

---

## ⚙️ 核心配置说明

主要配置文件位于 `src/config.ts`：

- **站点信息 (`siteConfig`)**：标题、副标题、默认语言（`zh_CN`）、默认主题色相、顶部横幅等。
- **个人名片 (`profileConfig`)**：姓名、签名、社交平台链接（GitHub, Twitter/X, Email 等）。
- **导航栏 (`navBarConfig`)**：顶部导航菜单项。

---

## 🌐 部署说明

### 1. GitHub Pages
项目已配置 `.github/workflows/deploy.yml`。每次向 `master` 分支执行 `git push`，GitHub Actions 会自动安装依赖、静态构建并更新 GitHub Pages。
> 请确保在 GitHub 仓库设置中的 **Settings -> Pages -> Build and deployment -> Source** 选择 **GitHub Actions**。

### 2. Cloudflare Pages
在 Cloudflare 控制台新建 Pages 项目，连接此 GitHub 仓库：
- **Framework preset**：`Astro`
- **Build command**：`pnpm build`
- **Build output directory**：`dist`
- **环境变量**：添加 `NODE_VERSION = 22`
