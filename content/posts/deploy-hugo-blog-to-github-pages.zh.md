---
title: '使用 Hugo + GitHub Pages 搭建博客'
date: 2026-07-06T11:30:00+08:00
draft: false
categories: ["技术"]
tags: ["Hugo", "GitHub Pages", "教程"]
summary: "一步步教你用 Hugo 搭建博客，通过 GitHub Actions 自动部署到 GitHub Pages。"
---

一步步教你用 Hugo 搭建博客，通过 GitHub Actions 自动部署到 GitHub Pages。

## 为什么选 Hugo + GitHub Pages

- **Hugo**：单二进制，构建极快，Go 语言生态
- **GitHub Pages**：免费托管，HTTPS，支持自定义域名
- **GitHub Actions**：零成本 CI/CD，推送即部署

---

## 1. 初始化 Hugo 站点

```bash
hugo new site blog --format yaml
cd blog
git init
```

添加主题（用 git submodule 管理）：

```bash
git submodule add https://github.com/adityatelange/hugo-PaperMod themes/PaperMod
```

编辑 `hugo.yaml`：

```yaml
baseURL: https://xyanrch1024.github.io
languageCode: zh-cn
title: xyanrch 的博客
theme: PaperMod
```

---

## 2. 双仓库策略

| 仓库 | 用途 | 你操作 |
|------|------|--------|
| `blog` | Hugo 源码（Markdown、主题、配置） | 是 — 在这里写文章 |
| `xyanrch1024.github.io` | 构建后的静态文件（HTML/CSS/JS） | 否 — Actions 自动管理 |

GitHub Pages 要求仓库名为 `<用户名>.github.io`，才能通过 `https://用户名.github.io/` 访问。

---

## 3. GitHub Actions 工作流

创建 `.github/workflows/deploy.yml`：

```yaml
name: Deploy to Pages
on:
  push:
    branches: [main]
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: true
      - uses: peaceiris/actions-hugo@v3
      - run: hugo --minify
      - uses: peaceiris/actions-gh-pages@v4
        with:
          personal_token: ${{ secrets.DEPLOY_TOKEN }}
          publish_dir: ./public
          external_repository: xyanrch1024/xyanrch1024.github.io
          publish_branch: main
```

关键点：
- `submodules: true` — 需要拉取 PaperMod 主题
- `personal_token` — 跨仓库部署必须用 PAT；`GITHUB_TOKEN` 不能推送到外部仓库
- `external_repository` — 目标仓库，即存放最终静态文件的仓库

---

## 4. Personal Access Token

`GITHUB_TOKEN`（自动生成）只能推送同仓库。要部署到 `xyanrch1024.github.io`，需要创建一个 classic PAT（`repo` 权限）：

1. 打开 https://github.com/settings/tokens
2. 生成 classic token，勾选 `repo` 权限
3. 在 blog 仓库添加 secret 名为 `DEPLOY_TOKEN`：

```bash
gh secret set DEPLOY_TOKEN --repo xyanrch1024/blog --body <你的token>
```

---

## 5. 启用 GitHub Pages

用户 Pages（`<用户名>.github.io`）本应自动启用，但有时需要推送一次才能触发首次构建。如果构建卡在 `building`，可以推送一个空提交：

```bash
git clone git@github.com:xyanrch1024/xyanrch1024.github.io.git
cd xyanrch1024.github.io
echo "trigger" >> trigger.txt
git add . && git commit -m "trigger rebuild" && git push
```

构建完成后（`status: built`），站点就在 `https://xyanrch1024.github.io/` 上线了。

---

## 6. 写作工作流

```bash
# 创建新文章
hugo new posts/my-article.md

# 本地预览
hugo server -D

# 部署
git add . && git commit -m "new article: xxx" && git push
```

推送触发 GitHub Action，自动构建 Hugo 并部署到 Pages。

---

## 总结

```
写文章 → 推送 → Actions 构建 → Pages 上线
```

- **本地**：`hugo new` + `hugo server -D`
- **推送**：`git push origin main`
- **CI**：Actions 自动构建和部署
- **托管**：GitHub Pages 提供静态文件服务

访问：https://xyanrch1024.github.io/
