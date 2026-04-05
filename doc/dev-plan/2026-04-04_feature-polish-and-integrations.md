# 2026-04-04 功能打磨 + 插件集成

## 本次目标

修复多个 UI/国际化问题，集成 hexo-blog-encrypt 密码保护，实现 Giscus 评论系统的亮暗模式实时同步。

## 完成步骤

### 1. Reflow 初始 Emoji 按亮暗模式区分

**需求：** 暗色模式下初始 emoji 为 🐰+🍓，亮色模式下为 🥛+🧊。

**改动文件：** `source/js/pretext-reflow.js`

启动时通过 `document.documentElement.classList.contains("dark")` 检测当前模式，据此初始化 `draggables` 数组。

**变量提升 bug：** 初版写了 `var dark = isDark()`，但 `isDark` 是后面才赋值的函数表达式。`var` 变量提升只提升声明不提升赋值，此时 `isDark` 为 `undefined`，调用 `undefined()` 抛出 TypeError，整个 IIFE 中止，canvas 完全不渲染。

修复为内联 `document.documentElement.classList.contains("dark")`，不依赖后续定义的变量。

详见 `debug-log/2026-04-04_darkmode-toggle-flicker.md`（同日闪烁修复的关联记录）。

### 2. Navbar 国际化修复

**问题：** 语言设为 `zh-CN` 时，导航栏仍显示英文 Home / Archives / Tags 等。

**根因：** `header.ejs` 遍历 `theme.menu` 对象时，直接用 key（`Home`、`Archives` 等）作为显示文本。语言文件中虽有 `menu.home: 首页` 等翻译，但模板没有调用 `__()` 查找。

**修复：** 将 `<%= label %>` 改为 `<%= __('menu.' + label.toLowerCase()) || label %>`。桌面端和移动端菜单各一处，共 2 处。

**改动文件：** `layout/_partial/header.ejs`

### 3. Pretext Reflow DOM 模式宽度对齐

**问题：** 切换到 DOM（Select Text）模式后，文章正文被 `max-w-content`（720px）约束，与上方 canvas 区域（`max-w-wide` = 1200px）以及 navbar 宽度不对齐，视觉上偏窄。

**修复：**
- `reflow-demo.ejs` 中 `#pretext-reflow-dom` 的 `max-w-content` 改为 `max-w-wide`，padding 改为 `px-4 sm:px-6 lg:px-8` 与 navbar 一致
- `pretext-reflow.ejs` 中 footer（标签、上下篇导航）同步从 `max-w-content` 改为 `max-w-wide`，加上相同 padding

**改动文件：** `layout/_partial/pretext/reflow-demo.ejs`、`layout/pretext-reflow.ejs`

### 4. 暗色模式切换闪烁修复

**问题：** 首页切换亮暗模式时，左侧文章列表文字出现 300ms 颜色渐变闪烁。

**根因：** `layout.ejs` 的 `<body>` 上有 `transition-colors duration-300`，所有继承 body 颜色的 DOM 元素同时触发 CSS transition。

**修复：** 移除 `transition-colors duration-300`。`<head>` 内联脚本已防止加载闪白，切换时瞬时变色更干净。

**改动文件：** `layout/layout.ejs`

详细分析见 `debug-log/2026-04-04_darkmode-toggle-flicker.md`。

### 5. 文章密码保护集成（hexo-blog-encrypt）

**方案选型：**

评估了两种方案后，选择「插件做加密 + 主题做样式适配」：

| | 自研加密 | hexo-blog-encrypt + 主题样式 |
|---|---|---|
| 加密逻辑 | 自己写，维护成本高 | 插件负责，社区验证 |
| UI 控制 | 完全自定义 | 通过 CSS 覆盖 |
| 职责分离 | 主题承担加密职责，越界 | 主题只管展示 |

**主题侧实现：**

在 `_tailwind.css` 中新增 `#hexo-blog-encrypt` 系列 CSS 覆盖：
- 输入框：JetBrains Mono 等宽字体，亮暗模式配色，focus 时边框变强调色
- 按钮：圆角药丸样式，hover 变色
- 摘要文本：Noto Serif JP 字体，居中显示
- 容器：`max-width: 720px` 居中

**居中对齐 bug：** 初版在 `#hexo-blog-encrypt` 容器上设了 `text-align: center`，解锁后文章内容继承居中。修复为只在 `.hbe-abstract` 和 `.hbe-input-container` 上设居中。

**文档更新：** `README.md` 新增「Password Protection / 文章密码保护」章节。

**改动文件：** `source/css/_tailwind.css`、`source/css/style.css`（重编译）、`README.md`

### 6. Giscus 评论系统亮暗模式同步

**原问题：** `giscus.ejs` 使用 `data-theme="preferred_color_scheme"`，跟随操作系统偏好而非页面实际亮暗状态。用户在暗色 OS 上手动切换到亮色主题时，giscus 仍显示深色，非常突兀。

**修复方案：**

1. **初始化时动态设置**：用 JS 创建 `<script>` 标签，根据页面当前 `dark` class 设置 `data-theme` 为 `'dark'` 或 `'light'`
2. **切换时实时同步**：通过 `MutationObserver` 监听 `<html>` 的 class 变化，变化时通过 `postMessage` 通知 giscus iframe 切换主题

**改动文件：** `layout/_partial/comment/giscus.ejs`（从静态 `<script>` 标签重写为动态 JS）

---

#### 技术讲解：postMessage 与 iframe 跨域通信

##### 为什么 Giscus 是 iframe

Giscus 的评论 UI 运行在 `giscus.app` 域名下，通过 `<iframe>` 嵌入到博客页面中。这是一个刻意的设计选择：

- **安全隔离**：评论内容（用户输入的 Markdown、HTML）在独立的源（origin）中渲染，即使含有恶意代码也无法访问宿主页面的 DOM、Cookie 或 localStorage
- **独立渲染**：评论区有自己的 CSS 和 JS，不会与博客主题的样式冲突
- **GitHub OAuth**：登录流程在 giscus.app 域内完成，博客站点无需处理 OAuth 令牌

##### 同源策略的限制

浏览器的同源策略（Same-Origin Policy）规定：不同源（协议 + 域名 + 端口）的页面之间**不能直接访问彼此的 DOM**。

```
博客页面 (example.com)          giscus iframe (giscus.app)
┌──────────────────┐           ┌──────────────────┐
│ document         │     ✗     │ document         │
│  └─ iframe       │ ────────→ │  └─ .giscus-frame│
│                  │  无法直接  │                  │
│                  │  操作 DOM  │                  │
└──────────────────┘           └──────────────────┘
```

这意味着博客页面**不能**直接执行 `iframe.contentDocument.querySelector('.theme').className = 'light'` 之类的操作。

##### postMessage：跨域通信的标准方式

`window.postMessage()` 是浏览器提供的安全跨域通信 API，绕过同源策略的 DOM 访问限制，让两个不同源的页面可以传递消息。

**发送端（博客页面）：**

```javascript
// 获取 giscus iframe 的 window 引用
var iframe = document.querySelector('iframe.giscus-frame');

// 向 iframe 发送消息，指定目标 origin
iframe.contentWindow.postMessage(
  { giscus: { setConfig: { theme: 'dark' } } },  // 消息数据（可以是任意可序列化的对象）
  'https://giscus.app'                             // 目标 origin（安全限制）
);
```

**接收端（giscus iframe 内部）：**

```javascript
// giscus 内部监听 message 事件
window.addEventListener('message', function(event) {
  // 1. 验证消息来源
  if (event.origin !== 'https://your-blog.com') return;

  // 2. 解析消息
  var data = event.data;
  if (data.giscus && data.giscus.setConfig) {
    // 3. 执行主题切换
    applyTheme(data.giscus.setConfig.theme);
  }
});
```

通信流程：

```
博客页面                          giscus iframe
┌──────────────┐                 ┌──────────────┐
│ 用户点击     │   postMessage   │              │
│ 暗色切换按钮 │ ──────────────→ │ 收到消息     │
│              │  {setConfig:    │ 验证 origin  │
│ MutationObs  │   {theme:      │ 切换主题 CSS │
│ 检测到 class │    'light'}}   │              │
│ 变化         │                 │              │
└──────────────┘                 └──────────────┘
```

##### Giscus 的消息协议

Giscus 定义了一套约定的消息格式，所有消息都包裹在 `{ giscus: { ... } }` 对象中：

```javascript
// 切换主题
{ giscus: { setConfig: { theme: 'light' } } }
{ giscus: { setConfig: { theme: 'dark' } } }

// 也支持自定义主题 CSS URL
{ giscus: { setConfig: { theme: 'https://example.com/custom.css' } } }
```

##### 安全注意事项

1. **发送时指定 targetOrigin**：`postMessage` 的第二个参数必须是目标 iframe 的确切 origin（`'https://giscus.app'`），而非 `'*'`。使用 `'*'` 会导致消息被发送到任意嵌入的 iframe，可能泄露敏感数据。

2. **接收时验证 event.origin**：接收端应始终检查 `event.origin` 是否为预期的来源，防止恶意页面伪造消息。

3. **消息数据不含敏感信息**：主题切换只传递 `'light'` / `'dark'` 字符串，不涉及令牌或用户数据。

##### 本主题的具体实现

```javascript
// 1. MutationObserver 监听 <html> 的 class 属性变化
new MutationObserver(function() {
  var dark = document.documentElement.classList.contains('dark');

  // 2. 找到 giscus 的 iframe
  var iframe = document.querySelector('iframe.giscus-frame');
  if (!iframe) return;

  // 3. 通过 postMessage 通知主题切换
  iframe.contentWindow.postMessage(
    { giscus: { setConfig: { theme: dark ? 'dark' : 'light' } } },
    'https://giscus.app'
  );
}).observe(document.documentElement, {
  attributes: true,
  attributeFilter: ['class']  // 只监听 class 变化，性能最优
});
```

这样，用户每次点击暗色模式切换按钮 → `<html>` 的 class 变化 → MutationObserver 触发 → postMessage 发送 → giscus iframe 收到并切换主题，整个链路在毫秒级完成，用户感知为即时同步。

---

### 7. Tailwind CSS 更新

- `_tailwind.css` 新增 `#hexo-blog-encrypt` 系列样式覆盖（输入框、按钮、摘要、容器）
- `style.css` 基于以上变更重新编译两次（密码保护初版 + 居中修复）

## 关键设计决策

| 决策 | 理由 |
|------|------|
| Reflow emoji 只在初始化时区分亮暗，切换后不替换 | 用户可能已拖拽/添加 emoji，强制替换会丢失交互状态 |
| Navbar 翻译用 `__()` fallback 到原始 label | 兼容用户自定义 menu key 名称不在语言文件中的情况 |
| 密码保护用 hexo-blog-encrypt 而非自研 | 加密非主题职责；社区插件更成熟；主题只需 CSS 覆盖 |
| `text-align: center` 限定在 `.hbe-abstract` 和 `.hbe-input-container` | 避免解密后正文继承居中 |
| Giscus 从 `preferred_color_scheme` 改为 JS 动态设置 | OS 偏好 ≠ 页面实际状态，手动切换时会不同步 |
| Giscus 主题同步用 postMessage 而非重载 iframe | postMessage 是毫秒级无闪烁切换；重载 iframe 会丢失已输入的评论内容 |
| 移除 body 的 `transition-colors` 而非缩短时长 | 大量小元素同时过渡本质上就会闪烁，瞬时切换更干净 |

## 当前新增/修改文件

| 文件 | 状态 | 说明 |
|------|------|------|
| `source/js/pretext-reflow.js` | 修改 | 初始 emoji 按亮暗模式区分 + 变量提升 bug 修复 |
| `layout/_partial/header.ejs` | 修改 | Navbar 国际化，`__()` 查翻译 |
| `layout/_partial/pretext/reflow-demo.ejs` | 修改 | DOM 层宽度放宽至 `max-w-wide` |
| `layout/pretext-reflow.ejs` | 修改 | footer 宽度放宽至 `max-w-wide` |
| `layout/layout.ejs` | 修改 | 移除 `transition-colors duration-300` |
| `source/css/_tailwind.css` | 修改 | 新增 hexo-blog-encrypt 样式覆盖 |
| `source/css/style.css` | 重新编译 | Tailwind 产物更新 |
| `README.md` | 修改 | 新增密码保护文档章节 |
| `layout/_partial/comment/giscus.ejs` | 重写 | 动态 theme + postMessage 同步 |
| `doc/debug-log/2026-04-04_darkmode-toggle-flicker.md` | 新增 | 闪烁修复分析 |
