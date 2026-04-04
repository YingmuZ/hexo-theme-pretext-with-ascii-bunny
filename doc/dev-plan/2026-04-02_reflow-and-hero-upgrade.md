# 2026-04-02 Pretext Reflow Demo + Hero 升级

## 本次目标

在主题中实现 pretext 库的核心展示场景：可拖拽 emoji 驱动的实时文字重排（reflow），以及首页 Hero 区域的 ASCII 兔子动画升级。同时补充测试站点内容，覆盖中英文、多分类/标签等场景。

## 完成步骤

### 1. 修复 post 页面崩溃 bug

- `article-full.ejs:21` 使用了 `stripHTML()`，不是 Hexo 内置 helper
- 改为 `strip_html()`，文章页恢复正常
- 详见 `debug-log/2026-04-02_post-page-crash.md`

### 2. 创建测试内容（hexo-test-site）

新增 5 篇长文章，覆盖多种排版场景：

| 文章 | 分类 | 标签 | 特点 |
|------|------|------|------|
| JavaScript 异步编程模式详解 | 技术笔记 | JavaScript, Async, 前端 | 大量代码块、表格 |
| CSS Grid 深入指南 | 技术笔记 | CSS, Grid, 布局 | photos 字段、代码块 |
| Rust 所有权机制 | 编程语言 | Rust, 系统编程, 内存安全 | 表格、引用块 |
| 设计系统中的排版规范 | 设计 | 设计系统, 排版, CSS | CSS 代码、排版理论 |
| 终端效率提升 | 工具箱 | 工具, 终端, 效率 | bash 代码块 |

新增 3 个页面：`/tags`、`/categories`、`/about`

### 3. Pretext Reflow Demo v1

创建 `layout/pretext-reflow.ejs` 专用布局 + `source/js/pretext-reflow.js` + `layout/_partial/pretext/reflow-demo.ejs`

核心实现：
- 通过 `layout: pretext-reflow` front-matter 指定，不影响普通文章
- 从 `page.content` 提取纯文本，通过 `data-text` 属性传入 Canvas
- CDN 动态 import `@chenglou/pretext`
- 🐰（48px）可拖拽，文字两侧环绕
- 排除区域计算：`circleHorizontalExclusion()` 计算每行与圆的水平交集
- 可用文字段：`buildAvailableSegments()` 从排除区域反推
- 逐段逐行排版：`layoutNextLine(prepared, cursor, segWidth)`

### 4. Reflow Demo v2 — 三项升级

**段落分隔：**
- EJS 中将 `</p><p>` 转为 `\n\n` 再 strip_html，保留段落结构
- JS 按 `\n\n` 拆分，每段独立 `prepareWithSegments`
- 段间插入 `LINE_HEIGHT × 1.2` 间距

**双 emoji 拖拽：**
- 单个 bunny 对象重构为 `draggables[]` 数组
- 🐰（48px, 半径 36）+ 🍓（36px, 半径 26），各自独立拖拽
- 多排除区域用 `mergeExclusions()` 合并重叠，再拆分可用段

**DOM/Canvas 切换修复：**
- 切换到 DOM 模式时，canvas 容器 `minHeight` 归零
- DOM 层用 `style="display:none"` 而非 Tailwind `hidden` class，避免优先级问题
- toggle 按钮改为 `sticky` 定位，跟随滚动

### 5. Hero 兔子 ASCII Art

替换原来的单行文字 Hero 为完整的兔子 ASCII Art（20 行）：

- 高度从 200px 扩大到 360px
- 兔子脸部区域（眼睛/鼻子）用朱红强调色
- 底部波浪线独立更大幅度波动
- `<noscript>` 和 fallback 都展示纯文本 `<pre>` 版兔子

**三档鼠标交互：**

| 档位 | 触发 | 振幅 | 速度 |
|------|------|------|------|
| idle | 鼠标不动（1.5s 超时） | 0.3px | 很慢 |
| wave | 鼠标在页面移动 | ~2px | 正常 |
| intense | hover 到 canvas 区域 | ~4px + 扩散 | 快 |

用 `currentIntensity` 变量在三档间 lerp 平滑过渡，动画函数读取 intensity 计算振幅/速度参数。

### 6. CJK 自动检测 + 中文长文章

- `pretext-reflow.js` 启动时统计 `data-text` 中 CJK 字符占比
- 超过 10% → 字体切换到 `Noto Serif SC`，行高从 28 调到 32
- 新增中文文章「文字排版的未来不在 CSS」（9 段落，`layout: pretext-reflow`）

### 7. Resize 闪烁修复

- ResizeObserver 回调中改为同步 `layoutAndRender()`
- 暗色模式切换、初始化同理改为同步
- 详见 `debug-log/2026-04-02_resize-flicker.md`

## 关键设计决策

| 决策 | 理由 |
|------|------|
| `layout: pretext-reflow` 专用布局，不侵入 post.ejs | 普通文章零影响，用户按需启用 |
| 段落独立 prepare，不合并为一段 | 保留段间距视觉节奏；避免超长文本一次性 prepare 的性能问题 |
| 双 emoji 排除区域先合并再拆分 | 两个 emoji 靠近时排除区域可能重叠，合并后才能正确计算可用段 |
| CJK 检测阈值 10% | 中英混排文章中，只要有一定比例中文就应该用衬线字体，10% 是保守阈值 |
| resize 同步渲染，drag 用 rAF | resize 会清空 canvas 必须同帧重绘；drag 不清空 canvas，rAF 节流更高效 |
| Hero 三档 lerp 过渡而非硬切换 | 避免动画突变，用户感知更自然 |

## 当前新增/修改文件

**hexo-pretext（主题仓库）：**
- `source/js/pretext-reflow.js` — 新增
- `source/js/pretext-hero.js` — 重写
- `layout/pretext-reflow.ejs` — 新增
- `layout/_partial/pretext/reflow-demo.ejs` — 新增
- `layout/_partial/pretext/hero.ejs` — 重写
- `layout/_partial/article-full.ejs` — bug fix（strip_html）

**hexo-test-site（测试站点）：**
- `source/_posts/` — 新增 7 篇文章（含 2 篇 reflow demo）
- `source/tags/index.md` — 新增
- `source/categories/index.md` — 新增
- `source/about/index.md` — 新增
