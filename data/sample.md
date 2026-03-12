# 欢迎使用 Markdown 预览器 🎉

这是一个简单的测试页面，用于展示你的 HTML 代码是否成功读取并渲染了 `data` 文件夹下的 Markdown 文件。

## 支持的特性

通过引入 `marked.js` 和 `github-markdown-css`，你可以获得非常棒的排版体验。

### 1. 文本格式化

你可以使用 **粗体**、*斜体*、~~删除线~~，以及 `内联代码`。

### 2. 列表

- 苹果 🍎
- 香蕉 🍌
- 橘子 🍊

有序列表：

1. 第一步：创建 `index.html`
2. 第二步：创建 `data/sample.md`
3. 第三步：启动本地服务器并预览

### 3. 代码块

支持语法高亮风格的代码块：

```
// 这是一个 JavaScript 代码示例
function sayHello(name) {
    console.log(`Hello, ${name}!`);
}
sayHello("World");
```

### 4. 引用

> 这是一个引用块。 "Talk is cheap. Show me the code." - Linus Torvalds

### 5. 表格

| 库名称              | 作用                       | 体积 |
| ------------------- | -------------------------- | ---- |
| marked.js           | 解析 Markdown 为 HTML      | 小巧 |
| github-markdown-css | 提供类似 GitHub 的页面排版 | 轻量 |

**测试成功！** 现在你可以随意修改这个 `sample.md` 文件的内容，刷新页面即可看到变化。