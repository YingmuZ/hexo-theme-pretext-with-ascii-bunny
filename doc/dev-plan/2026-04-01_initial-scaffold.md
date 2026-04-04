# 2026-04-01 Initial Scaffold — 主题骨架搭建

## 目标

从零搭建 hexo-theme-pretext 的完整文件结构，实现所有核心功能，使主题达到可运行状态。

## 完成步骤

### 1. 项目初始化
- 创建 `package.json`（声明 `hexo` 和 `hexo-renderer-ejs` 为 peerDependencies）
- 创建 `_config.yml` 主题默认配置（菜单、profile、pretext、评论、功能开关等）
- 创建 `tailwind.config.js`（自定义日式配色、字体、间距，`darkMode: 'class'`）
- 创建 `LICENSE`（MIT）

### 2. 基础布局
- `layout.ejs` — HTML 骨架，引入 head/header/footer，flex 布局 min-h-screen
- `head.ejs` — meta、Google Fonts preconnect、CSS 引入、暗色模式防闪烁内联脚本、GA
- `header.ejs` — 响应式导航栏 + 移动端汉堡菜单
- `footer.ejs` — 版权、powered by、社交链接
- `dark-mode-toggle.ejs` — 太阳/月亮图标切换按钮

### 3. 内容页面
- `index.ejs` — 首页，含 Pretext Hero 区域 + 文章列表 + 分页
- `post.ejs` — 文章详情 + 评论
- `page.ejs` — 独立页面
- `archive.ejs` — 按年分组归档列表
- `tag.ejs` — 标签索引 + 标签详情
- `category.ejs` — 分类索引 + 分类详情
- `article.ejs` — 列表页文章卡片（摘要 + Read More）
- `article-full.ejs` — 详情页完整文章（元信息、图片、正文、标签、上下篇导航）
- `pagination.ejs` — 上一页/下一页 + 页码
- `sidebar.ejs` — 个人资料 + 标签云
- `toc.ejs` — 文章目录（details/summary 折叠）

### 4. 评论系统
- `comment/index.ejs` — 根据 `theme.comments.provider` 动态加载
- `giscus.ejs` / `disqus.ejs` / `waline.ejs` — 三套评论系统
- Disqus 兼容 `config.disqus_shortname`（hexo-theme-unit-test 要求）

### 5. Pretext 集成
- 4 个 EJS 容器 partial（hero、post-ornament、archive-flow、404）
- 4 个 JS 文件，通过 CDN 动态 import `@chenglou/pretext`
- 使用的 API：`prepareWithSegments()`, `layoutWithLines()`, `prepare()`, `layout()`
- 每个场景均有 `<noscript>` + JS fallback

### 6. CSS & JS
- `_tailwind.css` — Tailwind directives + 自定义 base/components 层
- `style.css` — 编译产物（~13KB minified），已提交
- `main.js` — 暗色模式切换、移动菜单、回到顶部、代码复制按钮

### 7. i18n、Helpers、文档
- `zh-CN.yml` / `en.yml` — 完整中英文翻译
- `scripts/helpers.js` — `strip_html`、`reading_time` 辅助函数
- `README.md` — 中英双语安装与配置文档

## 关键设计决策

| 决策 | 理由 |
|------|------|
| Pretext 通过 CDN 加载，不打包进主题 | 保持主题体积极小，用户无需额外 npm install |
| 每个 Pretext 场景都有 CSS fallback | 渐进增强，JS 禁用时所有内容可读 |
| Disqus 同时支持 `config.disqus_shortname` 和 `theme.comments.disqus.shortname` | 兼容 hexo-theme-unit-test 要求 |
| `<head>` 内联同步暗色模式脚本 | 防止页面加载时闪白（FOUC） |
| Tailwind 编译产物 `style.css` 提交到仓库 | 用户安装后无需自己运行构建 |
| RSS `<link>` 在 head.ejs 中输出 | 兼容 hexo-generator-feed，支持 RSS autodiscovery |
| `<!DOCTYPE html>` 在 layout.ejs 首行 | hexo-theme-unit-test 合规项 |

## 当前文件总数

39 个文件（不含 node_modules）
