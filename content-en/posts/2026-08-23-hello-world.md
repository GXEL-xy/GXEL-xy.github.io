---
title: "Hello World — My First Blog Post"
date: 2026-08-24T13:20:00+08:00
lastmod: 2026-08-24T13:20:00+08:00
draft: false
description: "First test post, verifying the Markdown writing workflow"
tags: [hello, blog]
categories: [Notes]
---

Welcome to my blog! This is a sample post to verify the writing and publishing workflow of Hugo + Stack.

## Local Writing Workflow

1. Create a new Markdown file under `content-en/posts/`, named `YYYY-MM-DD-slug.md`
2. Fill in the Front Matter at the top (title, date, tags, etc.)
3. Preview locally with `hugo server`
4. Commit and push with git, and it will be published automatically

## Code Block Test

```python
def hello(name: str) -> str:
    return f"Hello, {name}!"
```

> Tip: Keep `draft: false` to publish; set it to `true` to keep the post as a draft.
