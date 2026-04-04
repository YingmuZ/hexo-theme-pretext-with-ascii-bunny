# 2026-04-03 首页重构 + 全局打磨

## 本次目标

重构首页为终端风格双栏布局，升级 Hero 为多兔子随机系统，完善 reflow demo 的动态交互，实现 post ornament 动画，并补充移动端触屏支持与样式组件。

## 完成步骤

### 1. 首页完全重构（index.ejs）

原首页使用 `article.ejs` partial 渲染文章卡片 + `pagination.ejs` 分页 + `sidebar.ejs` 侧边栏。本次彻底重写为双栏布局：

**左栏 — 终端风格文章列表：**
- 虚线边框容器，使用 box-drawing 字符装饰标题栏（`┌─ recent posts ─┐`）和底部（`└─────┘`）
- 仅展示最近 5 篇文章（`page.posts.toArray().slice(0, 5)`）
- 每篇文章一行式布局：`> YYYY-MM-DD │ 分类 / 子分类` + 标题 + `#标签`
- hover 时 `>` 前缀和标题同步变色（通过 `group` / `group-hover`）
- 底部 `> view all archives →` 链接

**右栏 — Hero 兔子动画：**
- 使用 `flex-1` 自适应剩余空间
- 移动端通过 `order-first` 让兔子显示在列表上方
- 桌面端 `lg:order-last` 兔子在右侧

**移除的依赖：**
- 不再引用 `article.ejs`、`pagination.ejs`、`sidebar.ejs` partial
- 分页逻辑移至 archives 页面承担

### 2. Hero 升级为多兔子随机系统（pretext-hero.js + hero.ejs）

**EJS 模板（hero.ejs）简化：**
- 移除固定 `360px` 高度，改为 `w-full h-full` 弹性布局
- 父容器由 `index.ejs` 控制尺寸（`flex items-center justify-center`）
- noscript / fallback 统一使用小型兔子 `<pre>`

**JS 端多兔子系统（pretext-hero.js）：**
- `BUNNIES[]` 数组包含 4 套不同风格 ASCII Art：
  1. Bug bunny（侧身趴姿，15 行）
  2. Standing bunny（正面站立长耳，21 行）
  3. Star bunny（星号轮廓，18 行）
  4. Sparkle bunny（侧身站立 + 星光装饰，17 行）
- 页面加载时 `Math.random()` 随机选取
- **字体自动缩放**：测量最宽行所需像素宽度，若超出容器则按比例缩小字号（最小 9px）
- **Canvas 高度动态计算**：`bunnyLines.length × scaledLineH + VERT_PAD × 2`，最小 200px，通过 `canvas.parentElement.style.height` 同步父容器
- 特殊字符着色逻辑不变：面部字符（O/o/0/@）用 accent 色，波浪线 `~` 和星光（`*`/`+`/`.`）用 secondary 色

### 3. Reflow Demo 动态添加 Emoji（reflow-demo.ejs + pretext-reflow.js）

**EJS 新增控件（reflow-demo.ejs）：**
- `+ emoji` 按钮（`#pretext-reflow-add`），与已有的 `📋 Select Text` 按钮并排
- 同为 sticky 定位 + `float-right`，跟随滚动

**JS 端实现（pretext-reflow.js）：**
- `EMOJI_POOL` — 10 种候选 emoji：🐰🍓🐑📖🥛🧊⛄🍀🦌🦷
- `MAX_DRAGGABLES = 10` — 上限保护
- `makeDraggable(emoji, fontSize, radius, x, y, showHint)` 工厂函数，统一创建可拖拽对象（含 glow 颜色、hint 文本等属性）
- 初始仍为 🐰（48px, r=36）+ 🍓（36px, r=26）两个
- 点击 `+ emoji` 时：
  1. 从池中排除已使用的 emoji，随机选取（全部用完则允许重复）
  2. 在内容区域随机位置生成（margin 60px 内缩）
  3. `draggables.push()` 后触发 `scheduleRender()`
- `updateAddBtn()` — 非 canvas 模式或达上限时禁用按钮（opacity + pointerEvents）

### 4. Post Ornament 实现（pretext-ornament.js + post-ornament.ejs）

**EJS 模板（post-ornament.ejs）：**
- 80×56 固定尺寸 Canvas，`flex-shrink-0` 防止被压缩
- noscript 和 fallback 都显示一个旋转 45° 的小方块

**JS 动画（pretext-ornament.js）：**
- ASCII 书本图案（5 行）：
  ```
   _______
  /      /,
  /      //
  /______//
  (______(/
  ```
- 逐字符解析为 `bookChars[]`，居中于 80×56 画布
- **shimmer 微动画**：
  - 位移：`sin(phase) × 0.4` / `cos(phase) × 0.3`，极小幅度晃动
  - alpha 呼吸：`0.75 + 0.25 × sin(...)`，慢速明暗交替
- 边缘字符（`/`, `(`, `)`, `,`）用 accent 色，其余用主色
- 通过 `prepareWithSegments()` 预热字体渲染（非关键，catch 忽略）

### 5. 404 页面增加触屏支持（pretext-404.js）

新增两个事件监听：
- `touchmove`（`passive: true`）：读取 `e.touches[0]` 坐标更新 mouseX/mouseY
- `touchend`：重置坐标为 -1000，字符回归原位

移动端用户可用手指触摸推开 ASCII Art 字符，与桌面端鼠标交互效果一致。

### 6. Tailwind / CSS 更新

**tailwind.config.js：**
- `fontFamily.body` 新增 `"Noto Serif SC"` 作为第二候选字体（`"Noto Serif JP"` 之后），改善简体中文渲染

**_tailwind.css 新增组件：**
- `.prose-excerpt`：摘要段落间距收紧（`margin-bottom: 0.75em`），末段无底部间距
- `.toc-nav`：目录导航样式 — 无序号、缩进 1em、hover 变色（亮色 `#3acce8`，暗色 `#E8563A`）

**style.css：**
- 基于以上变更重新编译 Tailwind 产物

## 关键设计决策

| 决策 | 理由 |
|------|------|
| 首页只展示 5 篇，不做分页 | 终端风格强调简洁，完整列表由 archives 承担；减少首页认知负荷 |
| 文章列表用 box-drawing 字符而非 CSS 边框 | 与主题的 ASCII / 终端美学统一，纯文本装饰感更强 |
| 多兔子随机而非用户配置选择 | 每次访问有新鲜感；不增加 `_config.yml` 配置复杂度 |
| Hero canvas 高度动态计算而非固定值 | 4 套兔子行数不同（15–21 行），固定高度会导致裁切或留白 |
| 字体自动缩放最小 9px | 在极窄屏幕（< 320px）下仍保证 ASCII Art 完整显示，低于 9px 已不可读 |
| Emoji 上限 10 个 | 超过 10 个排除区域计算开销显著增大，且视觉上文字已被挤压殆尽 |
| `makeDraggable()` 工厂函数 | 原来直接写对象字面量，新增动态添加后需要复用创建逻辑 |
| 404 触屏用 passive: true | touchmove 不需要 preventDefault，passive 避免滚动性能警告 |
| `Noto Serif SC` 加入 body 字体栈 | 与 reflow 的 CJK 检测呼应，确保非 reflow 页面也有良好的简中字体回退 |

## 当前新增/修改文件

| 文件 | 状态 | 说明 |
|------|------|------|
| `layout/index.ejs` | 重写 | 双栏终端风格首页 |
| `layout/_partial/pretext/hero.ejs` | 修改 | 去固定高度，改弹性布局 |
| `layout/_partial/pretext/post-ornament.ejs` | 修改 | 80×56 canvas 书本装饰 |
| `layout/_partial/pretext/reflow-demo.ejs` | 修改 | 新增 `+ emoji` 按钮 |
| `source/js/pretext-hero.js` | 重写 | 4 套兔子 + 自动缩放 + 动态高度 |
| `source/js/pretext-ornament.js` | 重写 | ASCII 书本 shimmer 动画 |
| `source/js/pretext-reflow.js` | 修改 | 动态添加 emoji + 工厂函数 |
| `source/js/pretext-404.js` | 修改 | 新增 touchmove/touchend 支持 |
| `tailwind.config.js` | 修改 | body 字体栈加入 Noto Serif SC |
| `source/css/_tailwind.css` | 修改 | 新增 prose-excerpt、toc-nav 组件 |
| `source/css/style.css` | 重新编译 | Tailwind 产物更新 |
