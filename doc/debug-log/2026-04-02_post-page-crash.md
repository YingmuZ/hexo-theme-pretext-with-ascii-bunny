# Post 页面渲染空白（0 bytes）

## 现象

点击首页 "Hello World" 文章链接，页面无法打开。`hexo generate` 后检查生成文件，`2026/04/02/hello-world/index.html` 为 0 bytes。

## 排查过程

1. `hexo generate` 输出中发现 ERROR，但静默生成了空文件
2. 查看完整错误信息：

```
ReferenceError: layout/_partial/article-full.ejs:21
stripHTML is not defined
```

3. 定位到 `article-full.ejs` 第 21 行：

```ejs
<span>&middot; <%= stripHTML(page.content).length %> words</span>
```

4. 确认 `stripHTML` 不是 Hexo 内置 helper

## 根因

EJS 模板中使用了 `stripHTML()`，但 Hexo 的内置 helper 名为 `strip_html()`（下划线命名）。EJS 渲染时抛出 ReferenceError，Hexo 捕获异常后输出空文件。

注意：虽然 `scripts/helpers.js` 中注册了一个自定义 `strip_html` helper，但 Hexo 本身已经内置了同名 helper，自定义版本实际上是冗余的。关键问题是模板中写的是 `stripHTML`（驼峰命名），两个版本都匹配不上。

## 修复

```diff
- <%= stripHTML(page.content).length %>
+ <%= strip_html(page.content).length %>
```

单行改动，文章页恢复正常（生成文件 12KB）。

## 经验总结

1. **Hexo helper 全部是 snake_case 命名**：`strip_html`、`trim`、`word_wrap` 等。在 EJS 中调用时不要想当然用驼峰。
2. **Hexo 渲染错误是静默的**：模板报错不会阻断 `hexo generate`，只会生成空文件。排查时应该关注 `hexo generate` 的 ERROR 输出，而不仅仅看是否有文件生成。
3. **开发时应该用 `hexo generate` 而不仅仅 `hexo server`**：server 模式下错误可能被浏览器吞掉（显示空白页），generate 模式下 ERROR 信息更明确。
