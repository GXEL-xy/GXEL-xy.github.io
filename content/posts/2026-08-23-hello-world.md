---
title: "你好，世界 —— 我的第一篇博客"
date: 2026-08-23T23:53:00+08:00
lastmod: 2026-08-23T23:53:00+08:00
draft: false
description: "第一篇测试文章，验证 Markdown 写作流程"
tags: [hello, 建站]
categories: [随笔]
---

欢迎来到我的博客！这是一篇示例文章，用于验证 Hugo + PaperMod 的写作与发布流程。

## 本地写作流程

1. 在 `content/posts/` 下新建 Markdown 文件，文件名格式：`YYYY-MM-DD-短标题.md`
2. 在文件头部填写 Front Matter（标题、日期、标签等）
3. 用 `hugo server` 本地预览
4. 写完后 `git add` + `git commit` + `git push`，自动发布

## 代码块测试

```python
def hello(name: str) -> str:
    return f"Hello, {name}!"
```

## 列表与引用

- 支持标签、分类归档
- 支持目录（TOC）
- 支持代码高亮与复制按钮

> 提示：发布正式内容前，把本文的 `draft: false` 保持即可；不想发布就改成 `true`。
