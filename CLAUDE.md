# ZijieGuo 技术博客 — 协作指南

## 维护约定

**每次对项目做出实质性修改后，必须同步更新 `AGENTS.md`。**
需要更新的内容包括但不限于：视觉主题调整、新增功能页面、关键文件变更、已知问题的发现或修复。
`AGENTS.md` 是面向所有 AI 代理的项目状态快照，保持它准确是协作的基础。

---

## 技术栈

- **框架**: Hugo (Extended)
- **主题**: PaperMod (git submodule at `themes/PaperMod/`)
- **部署**: GitHub Pages via `.github/workflows/deploy.yml`
- **URL**: https://zijieguo2003.github.io/blog

## 核心约定

**不要修改 `themes/PaperMod/` 内的任何文件。** 所有定制通过覆盖机制实现：

| 需求 | 做法 |
|---|---|
| 覆盖样式 | `assets/css/extended/custom.css` |
| 覆盖/新增模板 | `layouts/` 目录（镜像主题结构） |
| 静态资源 | `static/`（favicon、图片等） |
| 文章模板 | `archetypes/default.md` |

## 本地开发

```bash
hugo server -D          # 含草稿的预览，访问 http://localhost:1313/blog/
hugo --minify           # 检查构建是否成功
```

## 写作流程

```bash
hugo new posts/文章名/index.md   # bundle 格式（含图片）
hugo new posts/文章名.md         # 单文件格式
```

1. 编辑文章，完成后将 `draft: true` 改为 `draft: false`
2. `git add` → `git commit` → `git push`
3. GitHub Actions 自动构建并部署

## Frontmatter 规范

```yaml
---
title: '文章标题'
date: '2026-01-01T00:00:00+08:00'
lastmod: '2026-01-01T00:00:00+08:00'
author: "ZijieGuo"
description: "一句话描述（用于 SEO 和文章列表摘要）"
keywords: ["关键词1", "关键词2"]
tags: ["标签"]
categories: ["分类"]
draft: false
showToc: true
ShowReadingTime: true
ShowCodeCopyButtons: true
cover:
  image: "cover.png"   # 相对于文章目录（bundle 格式）或 static/
  alt: "封面描述"
  caption: ""
---
```

## 关键文件

- `hugo.toml` — 主配置（profileMode、搜索、菜单、社交链接）
- `assets/css/extended/custom.css` — 自定义样式（Claude 品牌配色：Pampas/Crail/Cloudy）
- `content/search.md` — 搜索页（layout: search）
- `content/archives.md` — 归档页（layout: archives）
- `static/images/avatar.jpg` — 首页头像（吉伊 Chiikawa）
- `static/favicon.jpg` — 站点图标（吉伊 Chiikawa）
- `AGENTS.md` — 项目状态快照，供 AI 代理上下文同步（每次修改后需维护）

## SEO 说明

- Sitemap: 自动生成，`/blog/sitemap.xml`
- robots.txt: 自动生成（`enableRobotsTXT = true`）
- OG / Twitter Card: PaperMod 主题内置
- `enableGitInfo = true` 让 `lastmod` 跟随 git 提交时间自动更新
