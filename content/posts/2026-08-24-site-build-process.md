---
title: "个人网站搭建完整过程"
date: 2026-08-24T14:58:00+08:00
lastmod: 2026-08-24T14:58:00+08:00
draft: false
description: "从零搭建 Hugo + Stack + GitHub Pages 个人网站的完整记录，含全部踩坑清单"
tags: [建站, Hugo, GitHub Pages]
categories: [教程]
---
# 个人网站搭建完整过程记录

> 记录时间:2026-08-23 ~ 2026-08-24
> 最终成果:https://GXEL-xy.github.io(基于 Hugo + Stack 主题 + GitHub Pages)

---

## 一、技术选型

| 项目 | 选择 | 理由 |
|---|---|---|
| 站点生成器 | **Hugo** | 单二进制、构建毫秒级、主题丰富、新手友好 |
| 主题 | **Stack**(CaiJimmy/hugo-theme-stack) | 现代卡片式、内置搜索/暗色模式、学术博客风格 |
| 托管 | **GitHub Pages**(GitHub Actions 自动部署) | 免费、免服务器、推代码即发布 |
| 域名 | GitHub 免费域名 `username.github.io` | 零成本,后续可绑自定义域名 |

---

## 二、环境搭建(踩坑实录)

### 2.1 安装 Hugo Extended

- 尝试 `winget install Hugo.Hugo.Extended` → **失败**:解压成功但创建符号链接时报 `create_symlink: The requested lookup key was not found`,winget 清理时把二进制也删了。
- **解决**:手动从 GitHub Releases 下载 `hugo_extended_0.165.0_windows-amd64.zip`,解压到 `C:\Users\86158\.workbuddy\binaries\hugo\`,并追加进用户 PATH。
- **坑**:`curl.exe` 是 Windows 程序,不识别 Git Bash 的 `/tmp` POSIX 路径 → 报 `curl: (23) Failure writing output to destination`。**解决**:下载输出路径用 Windows 风格路径(如 `C:\Users\...\Temp\...`)。

### 2.2 创建站点骨架

```bash
hugo new site . --force     # 在 D:\blog 生成骨架
git init -b main            # 初始化仓库(main 分支)
```

---

## 三、主题:从 PaperMod 到 Stack

### 3.1 第一版用 PaperMod

```bash
git submodule add https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
```

配置 `hugo.toml`(title、导航、homeInfoParams)。**坑**:TOML 中 `[menu.main]` 与 `[[menu.main]]` 混用会报 `key main already exists as a table`,正确写法是 `[menu]` 包裹 `[[menu.main]]`。

### 3.2 换用 Stack(用户最终选择)

```bash
git submodule add https://github.com/CaiJimmy/hugo-theme-stack.git themes/stack
```

**Stack v0.3+ 与 PaperMod 配置差异大,踩坑清单**:

| 配置项 | 错误写法 | 正确写法 |
|---|---|---|
| `colorScheme` | `colorScheme = "auto"`(字符串) | `[params.colorScheme] default="auto" toggle=true`(table) |
| `sidebar.avatar` | dict(`enabled=false`) | 字符串路径或 URL,留空不显示 |
| `widgets` | 字符串数组 | **对象数组**:`homepage = [{ type = "search" }, ...]` |
| 搜索功能 | — | 必须建 `content/search.md`(layout: search),数据来自搜索页自身 JSON,首页无需 JSON 输出 |
| 评论 | — | 主题默认开启(disqus),需 `[params.comments] enabled=false` 关闭 |
| `mainSections` | `["post"]`(主题默认) | `["posts"]`(对应 content/posts/) |

**坑**:后台 `hugo server` 运行时会锁定 `public/`,命令行构建报 `Access is denied` → 先 `Get-Process hugo | Stop-Process -Force` 停掉再构建。

---

## 四、部署上线 GitHub Pages

### 4.1 部署工作流 `.github/workflows/deploy.yml`

```yaml
on:
  push:
    branches: ["main"]
# 关键步骤:checkout(submodules: recursive) → peaceiris/actions-hugo
# → hugo --minify → upload-pages-artifact → deploy-pages
```

### 4.2 遇到的坑

| 坑 | 现象 | 解决 |
|---|---|---|
| **workflow 权限** | `remote rejected: refusing to allow a Personal Access Token to create or update workflow` | PAT 必须额外勾选 `workflow` scope(只勾 repo 不够) |
| **无 tty 无法认证** | `could not read Username for 'https://github.com'` | 非交互环境弹不出登录框;用带凭据 URL push,或 `git credential store` |
| **Pages 404** | 网站返回 404 | 仓库默认 Pages 是 legacy 模式(服务 main 分支源码),通过 API 切换:`PATCH /repos/{owner}/{repo}/pages {"build_type":"workflow"}` |
| **push 卡住** | `git push` 无响应 5 分钟 | 网络波动;改用带 token 的 URL push(不写入 remote 配置) |

### 4.3 用户名变更(15829959675 → GXEL-xy)

1. 用户网页操作:Settings → Admin → Change username。
2. 本地同步:改 `baseURL`、git 身份、remote URL、`~/.git-credentials`。
3. API 重命名仓库:`PATCH /repos/GXEL-xy/15829959675.github.io {"name":"GXEL-xy.github.io"}`。
4. 重新推送部署,验证新域名。

> 提示:改用户名后旧 `username.github.io` 会失效(旧用户名开放注册),新站是唯一地址。

---

## 五、内容与真实信息上线

- 站点标题 → `GXEL的个人网站`;侧边栏简介、头像、社交链接。
- **坑**:头像路径写了 Windows 绝对路径(`D:\blog\profile-photo\avataaars.png`)→ 线上 404。**正确**:图片放 `static/images/`,引用写站内路径 `/images/avatar.png`。
- 头像图片(海盗 avataaars 风格)上线成功。

---

## 六、功能扩展

### 6.1 Giscus 评论(commit bec0c99)

1. API 启用仓库 Discussions:`PATCH /repos/... {"has_discussions":true}`。
2. GraphQL 获取 `repo_id`(`R_kgDOUBr4Wg`)与分类 `category_id`(`DIC_kwDOUBr4Ws4DEDo6`,Announcements)。
3. 配置 `[params.comments] provider="giscus"`,mapping=`pathname`(每篇文章独立讨论帖)。
4. 用户还需:安装 giscus app(https://github.com/apps/giscus)授权给仓库。

### 6.2 GoatCounter 访问统计(commit e8e078e)

- Stack 主题无内置 Plausible/Umami → 用主题扩展点 `layouts/_partials/head/custom.html`(**站点 layouts 优先于主题**,不污染主题,更新主题不丢失)。
- 通过 `[params.goatcounter] enabled/code` 控制,用户注册 GoatCounter 后填入 code=`gxel` 激活。

### 6.3 中英双语(commit 6cb8d5d)

```toml
defaultContentLanguage = "zh-cn"
[languages.zh-cn]  # contentDir = "content",title/subtitle/菜单中文
[languages.en]     # contentDir = "content-en",英文版内容
```

- 英文内容目录 `content-en/`(首页、关于、搜索、示例文章)。
- 侧边栏出现语言切换按钮(`/en/` 前缀)。

### 6.4 侧边栏头像按天轮换(commit 3bb2b6d)

- 用户方案 A(JS 轮换):5 张图放 `static/images/`,JS 按 `dayOfYear % 5` 每天换一张。
- 脚本与统计共存于 `head/custom.html`。
- **坑**:Hugo `--minify` 会压缩内联 JS 变量名(`avatarList` → `e`),grep 原始变量名找不到属正常,验证看逻辑/路径即可。

---

## 七、坑点汇总(完整清单)

1. winget 装 Hugo 符号链接失败 → 手动下载二进制。
2. Windows 下 curl 不认 `/tmp` → 用 Windows 路径。
3. TOML 中 `[menu.main]`/`[[menu.main]]` 混用报错 → `[menu]` 包裹。
4. Stack 主题多项配置结构特殊(colorScheme/widgets/avatar/mainSections)。
5. `hugo server` 锁定 public → 先杀进程再构建。
6. PAT 推 workflow 文件需 `workflow` scope。
7. 无 tty 环境认证 → 带凭据 URL / credential store。
8. GitHub Pages legacy 模式 404 → API 切 `build_type: workflow`。
9. 头像/图片路径必须站内绝对路径 `/images/...`,禁止 Windows 路径。
10. `--minify` 压缩 JS 变量名,别用原变量名 grep 验证。
11. 推 workflow 后记得仓库 Pages 源选 GitHub Actions(或 API 切换)。

---

## 八、最终成果

- **地址**:https://GXEL-xy.github.io(中文默认,英文 `/en/`)
- **功能**:自动部署、评论(Giscus)、统计(GoatCounter)、中英双语、头像按天轮换、站内搜索、暗色模式、RSS
- **技术栈**:Hugo 0.165 extended + Stack 主题 + GitHub Actions + GitHub Pages
- **本地**:`D:\blog`,日常写作 `写 md → hugo server 预览 → git push → 自动发布`

---

*本记录与《网站操作手册》均保存在本地 `.workbuddy/` 目录,不入库不推送。*
