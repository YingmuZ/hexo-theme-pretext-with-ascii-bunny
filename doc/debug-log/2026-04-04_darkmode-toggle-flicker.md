# 首页切换亮暗模式时文字闪烁

## 现象

在首页点击暗色模式切换按钮时，左侧文章列表中的所有文本元素出现 300ms 的颜色渐变过渡，视觉上表现为集体闪烁。Hero canvas 区域不受影响。

## 根因分析

`layout.ejs` 的 `<body>` 标签上设置了 `transition-colors duration-300`：

```html
<body class="bg-bg-light dark:bg-bg-dark ... transition-colors duration-300">
```

切换暗色模式时，`document.documentElement` 上的 `dark` class 被 toggle，导致**所有继承 body 颜色的 DOM 元素**同时触发 300ms 的 CSS transition。

首页文章列表包含大量小文本元素（日期、分类、标签、标题），每个都在做独立的颜色过渡动画，集体效果就是闪烁。

Canvas 不受影响是因为 canvas 内容由 JS 直接绘制，不走 CSS transition。

## 修复

移除 `<body>` 上的 `transition-colors duration-300`：

```diff
- <body class="bg-bg-light dark:bg-bg-dark text-text-primary dark:text-text-primary-dark font-body leading-relaxed min-h-screen flex flex-col transition-colors duration-300">
+ <body class="bg-bg-light dark:bg-bg-dark text-text-primary dark:text-text-primary-dark font-body leading-relaxed min-h-screen flex flex-col">
```

## 为什么不需要过渡动画

1. `<head>` 中的内联脚本已经在 DOM 解析时同步设置 `dark` class，防止了页面加载时的白闪（FOUC）
2. 用户主动点击切换时，瞬时变色比 300ms 渐变更干净利落
3. 如果个别元素确实需要过渡效果（如 hover），应在该元素上单独设置，而非在 body 上全局施加
