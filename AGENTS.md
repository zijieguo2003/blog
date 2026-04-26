# ZijieGuo 技术博客 — 项目状态文档

> 本文件记录项目的**当前状态与关键设计决策**，供 AI 代理（Codex、Claude 等）接手时快速上下文同步。
> **每次对项目做出实质性修改后必须同步更新本文件。**

---

## 技术栈

| 组件 | 版本 / 说明 |
|---|---|
| 框架 | Hugo Extended ≥ 0.146.0（PaperMod 强制要求） |
| 主题 | PaperMod（git submodule，路径 `themes/PaperMod/`） |
| 部署 | GitHub Pages，workflow：`.github/workflows/deploy.yml` |
| 线上 URL | https://zijieguo2003.github.io/blog |

---

## 当前视觉主题（2026-04-26）

### 调色盘：Claude 品牌三色

| Token | 色号 | 名称 | 用途 |
|---|---|---|---|
| `--theme` | `#F4F3EE` | Pampas | 全站背景（页面、导航、列表页） |
| `--accent` | `#C15F3C` | Crail | 主强调色（链接、按钮、代码关键词） |
| `--secondary` | `#B1ADA1` | Cloudy | 次要文字、代码注释 |
| `--primary` | `#141413` | Dark | 主要文字 |
| `--entry` | `#FDFCF9` | — | 文章卡片背景 |
| `--border` | `#D8D5CC` | — | 分割线 |

### 代码块：Anthropic Light 风格（已确认）

| 变量 | 值 | 说明 |
|---|---|---|
| `--code-block-bg` | `#f4f1e8` | 浅暖奶油背景 |
| `--code-fg` | `#141413` | 深色文字 |
| `--code-keyword` | `#C15F3C` | Crail 关键词 |
| `--code-string` | `#059669` | 字符串 |
| `--code-number` | `#d97706` | 数字 |
| `--code-function` | `#0369a1` | 函数名 |
| `--code-type` | `#7c3aed` | 类型 |
| `--code-comment` | `#B1ADA1` | Cloudy 注释 |

### 字体

| 用途 | 字体 |
|---|---|
| 标题 (h1–h6) | Poppins → Arial → PingFang SC |
| 正文 | Lora → Georgia → PingFang SC |
| 代码 / 元信息 | Menlo → Consolas → Monaco |

### 图片资源

| 文件 | 内容 |
|---|---|
| `static/images/avatar.jpg` | 吉伊（Chiikawa）角色图，首页头像 |
| `static/favicon.jpg` | 吉伊（Chiikawa）角色图，站点图标 |

---

## 定制文件清单

所有定制**仅在以下文件中操作**，绝不修改 `themes/PaperMod/`：

| 文件 | 作用 |
|---|---|
| `assets/css/extended/custom.css` | 全部样式覆盖（颜色、字体、布局） |
| `layouts/partials/google_analytics.html` | GA stub（兼容旧版 Hugo） |
| `content/search.md` | 搜索页（`layout: search`） |
| `content/archives.md` | 归档页（`layout: archives`） |
| `archetypes/default.md` | 新文章模板 |
| `hugo.toml` | 站点主配置 |
| `static/` | 头像、favicon、静态资源 |

---

## 已知坑 & 覆盖说明

### 1. PaperMod 硬编码代码文字颜色
`themes/PaperMod/assets/css/common/md-content.css:158` 中有：
```css
.md-content pre code { color: rgb(213, 213, 214); }
```
这是为深色代码块设计的浅灰色，换成浅色代码背景后文字不可见。
**解决**：在 `custom.css` 中加：
```css
.md-content pre code { color: var(--code-fg) !important; }
```

### 2. 列表页背景与导航栏色差
PaperMod 对列表页（首页、归档等）设置 `background: var(--code-bg)`，与导航栏颜色不同导致头部"显白"。
**解决**：在 `custom.css` 中覆盖：
```css
.list { background: var(--theme); }
```

### 3. profileMode 头像路径
`hugo.toml` 中 `imageUrl = "images/avatar.jpg"`（相对路径，无前导 `/`）。
PaperMod 模板会用 `absURL` 拼接，在 GitHub Pages 子路径 `/blog` 下生成正确的完整 URL。
若加 `/` 前缀则图片路径错误。

### 4. Hugo 版本要求
PaperMod `theme.toml` 声明 `min_version = "0.146.0"`，低于此版本构建直接报错。
本地 Hugo 路径：`C:\Users\ZijieGuo\bin\hugo.exe`（手动安装 0.146.0 extended）。

### 5. 导航栏样式
`.header` 使用 `position: sticky; top: 0; z-index: 100` 实现吸顶。
背景色用 `var(--theme)`（Pampas），与页面一致，用 `border-bottom: 1px solid var(--border)` 作分隔线。

---

## 功能页面

| 页面 | URL | 说明 |
|---|---|---|
| 首页 | `/blog/` | profileMode（吉伊头像 + 标题 + 按钮） |
| 文章列表 | `/blog/posts/` | 文章卡片 |
| 归档 | `/blog/archives/` | 按年份归档 |
| 标签 | `/blog/tags/` | 标签云 |
| 搜索 | `/blog/search/` | Fuse.js 全文搜索 |

---

## 代码主题 Preset（可切换）

`custom.css` 中定义了四套可通过 `html[data-code-theme]` 切换的代码配色：

| Preset | 风格 |
|---|---|
| `ink` | 深色，暖棕基调 |
| `github` | 浅色，GitHub 官方风格 |
| `anthropic` | 浅色，Claude 品牌配色（与当前默认接近） |
| `one-dark` | 深色，Atom One Dark |

---

## 部署流程

```
git push master
  → GitHub Actions (.github/workflows/deploy.yml)
  → peaceiris/actions-hugo@v3 (hugo-version: latest)
  → hugo --minify
  → actions/deploy-pages@v4
  → https://zijieguo2003.github.io/blog
```
