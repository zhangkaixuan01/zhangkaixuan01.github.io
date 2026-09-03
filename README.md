# 个人技术博客

这是一个基于 Jekyll 的 GitHub Pages 博客。

## 本地写作

文章放在 `_posts/` 目录，文件名格式为 `YYYY-MM-DD-title.md`。提交并推送到 GitHub Pages 的发布分支后，网站会自动更新。

## 发布到 GitHub Pages

1. 在 GitHub 创建名为 `zhangkaixuan01.github.io` 的仓库。
2. 将本目录初始化为 Git 仓库并推送到该仓库的 `main` 分支。
3. 打开仓库 **Settings → Pages**，选择从 `main` 分支的根目录发布。

## 本地预览（可选）

安装 Ruby 和 Bundler 后，在项目目录运行：

```bash
gem install bundler jekyll
bundle exec jekyll serve
```

然后访问 `http://localhost:4000`。
