# 个人技术博客

这是一个基于 Hugo 和 PaperMod 的 GitHub Pages 技术博客。

## 本地写作

文章放在 `content/posts/` 目录，使用 Markdown 编写。主题配置位于 `hugo.toml`，PaperMod 以 Git 子模块保存在 `themes/PaperMod`。提交并推送到 `main` 分支后，GitHub Actions 会自动构建并更新网站。

## 发布到 GitHub Pages

1. 在 GitHub 创建名为 `zhangkaixuan01.github.io` 的仓库。
2. 将本目录推送到该仓库的 `main` 分支。
3. 打开仓库 **Settings → Pages**，将发布来源选择为 **GitHub Actions**。

## 本地预览（可选）

安装 Hugo 后，在项目目录运行（主题子模块需要已初始化）：

```bash
git submodule update --init --recursive
hugo server -D
```

然后访问 `http://localhost:4000`。
