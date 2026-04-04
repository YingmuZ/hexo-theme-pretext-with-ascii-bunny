# Pretext Reflow 页面 resize 时文字闪烁

## 现象

在 pretext-reflow-demo 页面（中文或英文），拖拽浏览器窗口改变尺寸时，Canvas 上的文字出现一闪一闪的效果，不够流畅。与 Cheng Lou 的 pretext 官方 demo 中丝滑的 resize 体验不符。

## 根因分析

Canvas API 的一个特性：**对 `canvas.width` 或 `canvas.height` 赋值会立即清空整个画布**。

原始代码流程：

```
ResizeObserver 触发
  → resize()
      → canvas.width = newWidth * dpr   ← 画布立即变空白
      → canvas.height = newHeight * dpr  ← 画布确认空白
      → ctx.setTransform(...)
  → scheduleRender()
      → requestAnimationFrame(callback)  ← 下一帧才重绘
```

问题出在 `resize()` 和 `scheduleRender()` 之间：

- `resize()` 执行完毕：画布已被清空
- `scheduleRender()` 注册的回调：要等到下一个 animation frame 才执行
- **这中间的一帧，用户看到的是空白画布** → 闪烁

当用户持续拖拽 resize 时，每帧都在重复「清空 → 等一帧 → 重绘 → 清空 → 等一帧 → 重绘」的循环，闪烁非常明显。

## 修复

将 ResizeObserver 回调中的 `scheduleRender()` 改为同步调用 `layoutAndRender()`：

```diff
  var ro = new ResizeObserver(function () {
    resize();
-   scheduleRender();
+   layoutAndRender(); // 同步：清空和重绘在同一帧内完成
  });
```

同理修复了暗色模式切换和初始化的路径：

```diff
  // 暗色模式切换
  new MutationObserver(function () {
-   scheduleRender();
+   layoutAndRender();
  })

  // 初始化
  prepareParagraphs();
  resize();
- scheduleRender();
+ layoutAndRender();
```

## 延伸：哪些路径该同步、哪些该用 rAF

| 场景 | 是否清空 canvas | 渲染方式 | 原因 |
|------|----------------|---------|------|
| resize（canvas 尺寸变化） | 是 | **同步** | 清空和重绘必须同帧 |
| 暗色模式切换 | 否，但颜色全变 | **同步** | 避免旧配色闪一帧 |
| 初始化首次绘制 | 是（canvas 刚创建） | **同步** | 用户不应看到空白 |
| 拖拽 emoji | 否 | **rAF 节流** | canvas 尺寸不变，不会清空；rAF 节流避免过度绘制 |
| canvas 高度自增长 | 是 | **同步**（递归调用） | 代码中已是同步递归，无问题 |

核心原则：**凡是涉及 `canvas.width/height` 赋值的路径，必须在同一帧内完成重绘。**

## 经验总结

1. **Canvas 的 width/height 赋值 = 隐式 clearRect**：这是 Canvas API 规范行为，不是 bug。很多 Canvas 开发新手会踩这个坑。
2. **ResizeObserver 回调中不要用 rAF 延迟渲染**：因为 ResizeObserver 本身已经在合适的时机触发（通常在 layout 之后、paint 之前），再加一层 rAF 反而多等一帧。
3. **rAF 节流适用于高频输入（mousemove/touchmove）**：这些事件触发频率可能高于屏幕刷新率，rAF 节流有意义。但 ResizeObserver 已经被浏览器节流过了，不需要再套 rAF。
4. **pretext 的 `layoutNextLine` 性能足够同步调用**：整篇文章的逐行排版在 1ms 以内完成，完全不需要分帧或异步处理。这正是 pretext 相比浏览器 reflow 的优势所在。
