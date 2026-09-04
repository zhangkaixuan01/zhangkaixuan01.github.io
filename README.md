# 张凯旋的技术博客

个人技术博客，记录 AI Agent、软件架构、工程实践与学习思考。

🌐 **在线阅读：** [zhangkaixuan01.github.io](https://zhangkaixuan01.github.io/)

## 技术栈

- [Hugo](https://gohugo.io/)：静态网站生成器
- [PaperMod](https://github.com/adityatelange/hugo-PaperMod)：博客主题
- GitHub Pages：网站托管
- GitHub Actions：自动构建与部署

## 项目结构

```text
content/posts/     # 博客文章
content/           # 关于、归档、搜索等页面
hugo.toml          # Hugo 与主题配置
themes/PaperMod/   # PaperMod 主题（Git 子模块）
.github/workflows/ # 自动部署工作流
```

## 写作

在 `content/posts/` 下新建 Markdown 文件，例如：

```text
content/posts/my-first-post.md
```

文章需要包含 Hugo front matter，例如：

```markdown
---
title: "文章标题"
date: 2026-09-04T12:00:00+08:00
description: "文章摘要"
tags: [Hugo, 技术]
draft: false
---

正文内容。
```

提交并推送到 `main` 分支后，GitHub Actions 会自动构建并发布网站。

## 本地预览

安装 Hugo 后，在项目目录执行：

```bash
git clone --recurse-submodules https://github.com/zhangkaixuan01/zhangkaixuan01.github.io.git
cd zhangkaixuan01.github.io
hugo server -D
```

然后访问 <http://localhost:1313/>。

## 主题说明

本博客使用 PaperMod 主题。主题通过 Git 子模块引入，更新主题时可执行：

```bash
git submodule update --remote --merge
```

欢迎通过 Issue 或 Pull Request 提出建议。
