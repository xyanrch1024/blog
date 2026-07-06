---
title: 'Deploy Hugo Blog to GitHub Pages'
date: 2026-07-06T11:30:00+08:00
draft: false
summary: "A step-by-step guide to setting up a Hugo blog with automatic deployment to GitHub Pages using GitHub Actions."
---

*A step-by-step guide to setting up a Hugo blog with automatic deployment to GitHub Pages using GitHub Actions.*

## Why Hugo + GitHub Pages

- **Hugo**: single binary, extremely fast build, Go ecosystem
- **GitHub Pages**: free hosting, HTTPS, custom domain support
- **GitHub Actions**: zero-cost CI/CD, push-to-deploy workflow

---

## 1. Initialize Hugo Site

```bash
hugo new site blog --format yaml
cd blog
git init
```

Add a theme as a git submodule:

```bash
git submodule add https://github.com/adityatelange/hugo-PaperMod themes/PaperMod
```

Edit `hugo.yaml`:

```yaml
baseURL: https://xyanrch1024.github.io
languageCode: zh-cn
title: xyanrch 的博客
theme: PaperMod
```

---

## 2. Two-Repo Strategy

| Repo | Purpose | You Touch |
|------|---------|-----------|
| `blog` | Hugo source (markdown, theme, config) | Yes — write articles here |
| `xyanrch1024.github.io` | Built static files (HTML/CSS/JS) | No — managed by Actions |

GitHub Pages requires the repo name to be `<username>.github.io` to serve at `https://username.github.io/`.

---

## 3. GitHub Actions Workflow

Create `.github/workflows/deploy.yml`:

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

Key points:
- `submodules: true` — needed for the PaperMod theme
- `personal_token` — required for cross-repo deployment; `GITHUB_TOKEN` cannot push to external repos
- `external_repository` — target repo where the built site lives

---

## 4. Personal Access Token

`GITHUB_TOKEN` (auto-generated) can only push within the same repo. To deploy to `xyanrch1024.github.io`, create a classic PAT with `repo` scope:

1. Go to https://github.com/settings/tokens
2. Generate a classic token with `repo` scope
3. Add it as a secret named `DEPLOY_TOKEN` in the blog repo:

```bash
gh secret set DEPLOY_TOKEN --repo xyanrch1024/blog --body <your-token>
```

---

## 5. Enable GitHub Pages

For user pages (`<username>.github.io`), GitHub Pages is auto-enabled but sometimes needs a push to trigger the first build. If the build gets stuck at `building`, push a dummy commit:

```bash
git clone git@github.com:xyanrch1024/xyanrch1024.github.io.git
cd xyanrch1024.github.io
echo "trigger" >> trigger.txt
git add . && git commit -m "trigger rebuild" && git push
```

After the build completes (`status: built`), the site goes live at `https://xyanrch1024.github.io/`.

---

## 6. Writing Workflow

```bash
# Create a new post
hugo new posts/my-article.md

# Preview locally
hugo server -D

# Deploy
git add . && git commit -m "new article: xxx" && git push
```

Push triggers the GitHub Action, which builds Hugo and deploys to Pages automatically.

---

## Summary

```
Write → Push → Actions build → Pages live
```

- **Local**: `hugo new` + `hugo server -D`
- **Git**: `git push origin main`
- **CI**: Actions builds and deploys
- **Hosting**: GitHub Pages serves the static site

Visit: [https://xyanrch1024.github.io/](https://xyanrch1024.github.io/)
