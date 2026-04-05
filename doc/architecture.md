# hexo-theme-pretext 技术架构文档

## 1. 项目总览

hexo-theme-pretext 是一个 Hexo **主题**，不是 Hexo 站点。两者的关系：

```
hexo-test-site/                  ← Hexo 站点（用户的博客）
├── _config.yml                  ← 站点配置（config.xxx）
├── _config.pretext.yml          ← 主题配置覆盖（theme.xxx，Hexo 5+）
├── source/_posts/               ← 用户的文章
├── themes/
│   └── pretext/                 ← 本项目（主题）
│       ├── _config.yml          ← 主题默认配置（theme.xxx）
│       ├── layout/              ← EJS 模板
│       ├── source/              ← 静态资源（CSS、JS）
│       ├── scripts/             ← Hexo helper 脚本
│       └── languages/           ← 翻译文件
└── node_modules/
```

**职责边界：**
- 站点负责：内容（文章、页面）、部署配置、插件安装
- 主题负责：展示（模板、样式、交互动画）

用户可以通过 npm 安装主题（`node_modules/hexo-theme-pretext`），也可以 clone 到 `themes/pretext`，Hexo 两种方式都支持。

## 2. 目录结构速查

```
hexo-theme-pretext/
├── _config.yml              主题默认配置，定义所有可配置项及默认值
├── package.json             声明 peerDependencies（hexo、hexo-renderer-ejs）
├── tailwind.config.js       Tailwind CSS 构建配置
├── LICENSE                  MIT 许可证
├── README.md                安装与使用文档
│
├── layout/                  EJS 模板（Hexo 渲染入口）
│   ├── layout.ejs           HTML 骨架（<html><head><body>），所有页面共用
│   ├── index.ejs            首页（终端风格文章列表 + Hero）
│   ├── post.ejs             文章详情页
│   ├── page.ejs             独立页面（About 等）
│   ├── archive.ejs          归档列表
│   ├── tag.ejs              标签索引 / 标签详情
│   ├── category.ejs         分类索引 / 分类详情
│   ├── pretext-reflow.ejs   Pretext Reflow 专用布局
│   └── _partial/            可复用组件（partial）
│       ├── head.ejs         <head> 内容（meta、字体、CSS、暗色模式脚本）
│       ├── header.ejs       导航栏
│       ├── footer.ejs       页脚
│       ├── dark-mode-toggle.ejs   暗色模式切换按钮
│       ├── article.ejs      文章卡片（列表页用）
│       ├── article-full.ejs 完整文章（详情页用）
│       ├── sidebar.ejs      侧边栏
│       ├── toc.ejs          目录
│       ├── pagination.ejs   分页导航
│       ├── comment/         评论系统
│       │   ├── index.ejs    路由分发
│       │   ├── giscus.ejs   Giscus 嵌入
│       │   ├── disqus.ejs   Disqus 嵌入
│       │   └── waline.ejs   Waline 嵌入
│       └── pretext/         Pretext 动画容器
│           ├── hero.ejs     首页兔子动画
│           ├── 404.ejs      404 页面 ASCII Art
│           ├── post-ornament.ejs  文章装饰图案
│           ├── archive-flow.ejs   归档页动画容器
│           └── reflow-demo.ejs    Reflow 交互演示
│
├── source/                  静态资源（Hexo 会复制到生成目录）
│   ├── css/
│   │   ├── _tailwind.css    Tailwind 源文件（开发者编辑这个）
│   │   └── style.css        Tailwind 编译产物（浏览器加载这个）
│   └── js/
│       ├── main.js          通用功能（暗色切换、移动菜单、回到顶部、代码复制）
│       ├── pretext-hero.js  首页 Hero 兔子动画
│       ├── pretext-404.js   404 页面交互动画
│       ├── pretext-ornament.js  文章装饰 shimmer 动画
│       ├── pretext-archive.js   归档页折叠动画
│       └── pretext-reflow.js    文字重排交互引擎
│
├── scripts/
│   └── helpers.js           注册 Hexo helper（strip_html、reading_time）
│
├── languages/
│   ├── en.yml               英文翻译
│   └── zh-CN.yml            简体中文翻译
│
└── doc/
    ├── architecture.md      本文档
    ├── dev-plan/            开发日志（按日期记录变更）
    └── debug-log/           Bug 修复记录
```

## 3. Hexo 渲染流程

用户运行 `hexo generate`（或 `hexo server`）时的完整流程：

### 3.1 Markdown → HTML

```
source/_posts/my-post.md
        │
        ▼
┌─────────────────────────┐
│ hexo-renderer-marked    │  Hexo 内置的 Markdown 渲染器
│ （或其他 Markdown 插件） │  将 Markdown 转为 HTML 片段
└─────────────────────────┘
        │
        ▼
  HTML 片段（page.content）
```

此时 `page.content` 只是文章正文的 HTML，没有 `<html>`、`<head>`、导航栏等。

### 3.2 EJS 模板注入

Hexo 根据页面类型选择对应的 layout 文件：

```
首页    → layout/index.ejs
文章    → layout/post.ejs
页面    → layout/page.ejs
归档    → layout/archive.ejs
标签    → layout/tag.ejs
分类    → layout/category.ejs
```

选中的 layout 文件渲染后成为 `body` 变量，然后注入到 `layout.ejs`：

```
layout.ejs（HTML 骨架）
├── <%- partial('_partial/head') %>     → <head> 内容
├── <%- partial('_partial/header') %>   → 导航栏
├── <%- body %>                         → index/post/page 等的渲染结果
├── <%- partial('_partial/footer') %>   → 页脚
└── <%- js('js/main.js') %>             → 通用 JS
```

### 3.3 特殊布局

文章可以在 front-matter 中指定非默认布局：

```yaml
---
layout: pretext-reflow    # 使用 layout/pretext-reflow.ejs 而非 post.ejs
---
```

此时 Hexo 会用 `pretext-reflow.ejs` 替代 `post.ejs` 作为 `body`，但仍然注入到 `layout.ejs` 骨架中。

### 3.4 `config` vs `theme`

EJS 模板中有两个全局变量来源：

| 变量 | 来源 | 示例 |
|------|------|------|
| `config.xxx` | 站点 `_config.yml` | `config.title`、`config.language`、`config.url` |
| `theme.xxx` | 主题 `_config.yml`（被 `_config.pretext.yml` 覆盖） | `theme.menu`、`theme.comments`、`theme.pretext` |

Hexo 5+ 的合并优先级：`_config.pretext.yml` > `themes/pretext/_config.yml`。

这就是为什么评论配置要放在 `_config.pretext.yml` 而非站点 `_config.yml` 中 —— 模板读的是 `theme.comments`，只有主题配置层级的文件才会被映射到 `theme` 对象。

## 4. CSS 构建流程（Tailwind）

### 4.1 两个文件的关系

```
_tailwind.css（源文件）          style.css（编译产物）
┌───────────────────┐           ┌───────────────────┐
│ @tailwind base;   │           │ *, ::before, ...  │
│ @tailwind components;         │ .prose p { ... }  │
│ @tailwind utilities;          │ .max-w-wide { ... }│
│                   │  ──────→  │ .dark .prose th {  │
│ @layer components {│ npx      │   ... }           │
│   .prose p { ... }│ tailwindcss│ .hover\:text-     │
│   ...             │           │   accent:hover {  │
│ }                 │           │   ... }           │
└───────────────────┘           └───────────────────┘
  开发者编辑这个                   浏览器加载这个
  不会被 Hexo 处理                 Hexo 原样复制到 public/
```

### 4.2 编译命令

```bash
npx tailwindcss -i source/css/_tailwind.css -o source/css/style.css --minify
```

- `-i`：输入（源文件）
- `-o`：输出（产物）
- `--minify`：压缩输出

### 4.3 Tailwind 如何知道用了哪些 class

`tailwind.config.js` 中的 `content` 字段告诉 Tailwind 去扫描哪些文件：

```javascript
content: ["./layout/**/*.ejs"]
```

Tailwind 会解析所有 EJS 模板，提取出用到的 class 名（如 `max-w-wide`、`dark:bg-bg-dark`、`hover:text-accent`），**只有被使用的 class 才会出现在 `style.css` 中**。这就是为什么编译后的 CSS 只有 ~40KB 而非 Tailwind 完整库的数 MB。

### 4.4 什么时候需要重新编译

| 改动类型 | 需要重新编译？ | 原因 |
|---------|--------------|------|
| 修改 `_tailwind.css`（自定义样式） | **是** | 源文件变了，产物需要更新 |
| 在 EJS 中使用了新的 Tailwind class | **是** | 新 class 不在当前产物中 |
| 修改 `tailwind.config.js`（颜色、字体等） | **是** | 配置变了，生成的 utility class 值会变 |
| 修改 EJS 模板但没用新 class | **否** | 已有的 class 都在 style.css 中 |
| 修改 JS 文件 | **否** | JS 不走 Tailwind 构建 |
| 修改 `_config.yml` | **否** | 配置文件不影响 CSS |

### 4.5 为什么 style.css 提交到仓库

用户安装主题后应该**开箱即用**，不需要自己安装 Tailwind 或运行构建命令。`style.css` 作为编译产物提交到仓库，Hexo 直接将其复制到 `public/css/style.css`，浏览器直接加载。

`_tailwind.css` 和 `tailwind.config.js` 只有主题开发者需要关心。`devDependencies` 中的 `tailwindcss`、`postcss`、`autoprefixer` 也只在开发时使用。

### 4.6 文件名前缀 `_` 的含义

`_tailwind.css` 以下划线开头。Hexo 默认会将 `source/` 下的文件复制到 `public/`，但以 `_` 开头的文件会被 Hexo 忽略。这样 `_tailwind.css`（源文件）不会出现在生成的站点中，只有 `style.css`（产物）会被复制。

## 5. Pretext 集成架构

### 5.1 加载方式

所有 Pretext 场景通过 CDN 动态 import 加载，不打包进主题：

```javascript
var mod = await import("https://cdn.jsdelivr.net/npm/@chenglou/pretext/+esm");
var prepareWithSegments = mod.prepareWithSegments;
```

**为什么不打包：**
- 保持主题体积极小（用户 npm install 只下载主题代码）
- Pretext 是渐进增强层，加载失败不影响内容可读性
- CDN 有全球缓存，加载速度通常优于从博客服务器获取

### 5.2 五个场景

| 场景 | EJS 容器 | JS 文件 | 使用的 API |
|------|---------|---------|-----------|
| Hero 兔子 | `pretext/hero.ejs` | `pretext-hero.js` | `prepareWithSegments()` |
| 404 页面 | `pretext/404.ejs` | `pretext-404.js` | `prepareWithSegments()` + `layoutWithLines()` |
| 文章装饰 | `pretext/post-ornament.ejs` | `pretext-ornament.js` | `prepareWithSegments()` |
| 归档动画 | `pretext/archive-flow.ejs` | `pretext-archive.js` | 纯 DOM 动画（不用 pretext API） |
| 文字重排 | `pretext/reflow-demo.ejs` | `pretext-reflow.js` | `prepareWithSegments()` + `layoutNextLine()` |

### 5.3 渐进增强三层

每个 Pretext 场景都有三层渲染策略：

```
层级 1：<noscript>
  └─ 纯 HTML/CSS 展示（JS 完全禁用时）

层级 2：JS fallback
  └─ Pretext CDN 加载失败时，显示 CSS 静态替代

层级 3：Canvas 动画
  └─ Pretext 加载成功，完整交互体验
```

EJS 模板中的典型结构：

```html
<canvas id="pretext-hero-canvas"></canvas>

<noscript>
  <pre>ASCII Art 纯文本</pre>
</noscript>

<div id="pretext-hero-fallback" class="hidden">
  <pre>ASCII Art 纯文本</pre>
</div>
```

JS 中的降级逻辑：

```javascript
try {
  var mod = await import("https://cdn.jsdelivr.net/npm/@chenglou/pretext/+esm");
  // 成功 → 启动 Canvas 动画
} catch {
  // 失败 → 隐藏 canvas，显示 fallback
  canvas.style.display = "none";
  fallback.classList.remove("hidden");
}
```

### 5.4 Canvas 渲染 vs DOM 渲染

Reflow Demo 支持两种模式切换：

| | Canvas 模式 | DOM 模式 |
|---|---|---|
| 显示 | `<canvas>` 绘制文字 | 原始 `page.content` HTML |
| 交互 | 可拖拽 emoji，文字实时重排 | 标准文本选择、复制 |
| 切换按钮 | `📋 Select Text` | `✨ Interactive` |
| 实现 | `pretext-reflow.js` 逐行排版 | 浏览器原生渲染 |

两层在 DOM 中始终存在，通过 `display: none` / `display: block` 切换，避免重复创建。

## 6. 暗色模式实现

### 6.1 整体架构

暗色模式基于 `class` 策略（`tailwind.config.js` 中 `darkMode: "class"`），由 `<html>` 元素上的 `dark` class 控制：

```html
<html class="dark">  <!-- 暗色模式 -->
<html class="">      <!-- 亮色模式 -->
```

### 6.2 防闪烁（FOUC Prevention）

`head.ejs` 中有一段**内联同步脚本**，在 `<body>` 渲染之前执行：

```javascript
(function(){
  var t = localStorage.getItem('theme');
  if (t === 'dark' || (!t && window.matchMedia('(prefers-color-scheme: dark)').matches)) {
    document.documentElement.classList.add('dark');
  }
})();
```

**为什么必须内联同步：** 如果用外部 JS 文件（`<script src="...">`），浏览器会先渲染一帧亮色页面，等 JS 加载完再切到暗色，造成白色闪烁（Flash of Unstyled Content）。内联同步脚本在 DOM 构建时立即执行，保证第一帧就是正确的颜色。

**优先级：** localStorage 手动选择 > OS 系统偏好 > 默认亮色。

### 6.3 切换逻辑

`main.js` 中的切换按钮：

```javascript
toggle.addEventListener("click", function () {
  var isDark = html.classList.contains("dark");
  html.classList.toggle("dark");             // 切换 class
  localStorage.setItem("theme", isDark ? "light" : "dark");  // 持久化
});
```

### 6.4 各子系统如何响应

| 子系统 | 响应机制 |
|--------|---------|
| CSS（Tailwind） | `dark:` 前缀自动生效（如 `dark:bg-bg-dark`），由浏览器 CSS 引擎处理 |
| Pretext Canvas | 每帧读取 `isDark()` 函数获取当前颜色，无需额外监听 |
| Pretext Reflow | `MutationObserver` 监听 `<html>` class 变化 → 同步重新渲染 |
| Giscus 评论 | `MutationObserver` 监听 → `postMessage` 通知 iframe 切换主题 |

## 7. 国际化（i18n）

### 7.1 语言文件结构

```yaml
# languages/zh-CN.yml
menu:
  home: 首页
  archives: 归档
post:
  read_more: 阅读全文
  posted_on: 发布于
pagination:
  prev: 上一页
  next: 下一页
```

### 7.2 使用方式

EJS 模板中通过 `__()` helper 查找翻译：

```ejs
<%= __('menu.home') %>        <%# 输出「首页」（zh-CN）或「Home」（en） %>
<%= __('post.read_more') %>   <%# 输出「阅读全文」或「Read More」 %>
```

### 7.3 语言选择

由站点 `_config.yml` 的 `language` 字段决定：

```yaml
language: zh-CN    # 使用 languages/zh-CN.yml
language: en       # 使用 languages/en.yml
```

Hexo 会在主题的 `languages/` 目录中查找对应文件名的 YAML。

### 7.4 Fallback 策略

模板中通常写 `<%= __('menu.home') || 'Home' %>`，当翻译缺失时 fallback 到英文硬编码。这保证即使用户使用了未提供翻译的语言，页面也不会显示空白。

## 8. 评论系统

### 8.1 路由分发

`comment/index.ejs` 根据 `theme.comments.provider` 决定加载哪个评论系统：

```
theme.comments.provider
├── 'giscus'  → partial('giscus')
├── 'disqus'  → partial('disqus')
├── 'waline'  → partial('waline')
└── false     → 不渲染评论区
```

### 8.2 三种 provider 的技术差异

| | Giscus | Disqus | Waline |
|---|---|---|---|
| 载体 | iframe | 动态 `<script>` | ESM import |
| 数据存储 | GitHub Discussions | Disqus 服务器 | 自部署服务端 |
| 暗色模式 | `postMessage` 同步 | Disqus 自身控制 | `dark: 'html.dark'` CSS 选择器 |
| 登录方式 | GitHub OAuth（在 iframe 内） | Disqus 账号 | 匿名或自定义 |
| 加载方式 | `<script>` 创建 iframe | `<script>` 注入 DOM | `import()` 初始化 |

### 8.3 Giscus 亮暗模式同步细节

Giscus 运行在跨域 iframe 中，父页面无法直接操作其 DOM。主题通过以下链路实现实时同步：

```
用户点击暗色按钮
  → main.js toggle <html> class
  → MutationObserver 检测到 class 变化
  → postMessage 发送 { giscus: { setConfig: { theme: 'dark' } } }
  → giscus iframe 接收消息并切换主题
```

详细的 postMessage 技术讲解见 `dev-plan/2026-04-04_feature-polish-and-integrations.md` 第 6 节。

## 9. 插件集成点

主题本身不包含任何 Hexo 插件，但提供了与以下插件的兼容设计：

### 9.1 hexo-blog-encrypt

- **插件职责**：构建时加密文章内容，运行时提供密码输入和解密 UI
- **主题职责**：通过 CSS 覆盖（`#hexo-blog-encrypt` 选择器）让密码输入 UI 匹配主题风格
- **集成方式**：纯 CSS，零模板改动，零 JS 改动

### 9.2 hexo-generator-feed

- **插件职责**：生成 `atom.xml` RSS 文件
- **主题职责**：在 `head.ejs` 中输出 `<link rel="alternate" type="application/atom+xml">`，支持 RSS autodiscovery
- **配置**：`theme.rss` 指向 feed 路径（默认 `/atom.xml`）

### 9.3 hexo-renderer-ejs

- **这是必需依赖**（peerDependency），不是可选插件
- Hexo 通过它来渲染 `.ejs` 模板文件
- 用户必须安装：`npm install hexo-renderer-ejs`
