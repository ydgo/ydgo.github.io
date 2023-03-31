---
title: "基于 Github Action 自动构建 Hugo 博客"
date: 2023-03-30T16:19:41+08:00
draft: false
categories: ["Blog"]
tags: [Hugo]
---

记录如何使用 Hugo 和 Github Action 自动部署博客系统。

<!--more-->

## 1. 创建 Github 项目
1. 项目名称符合规范：`username.github.io`
2. 配置好仓库的配置页面的 Pages 配置参数

## 2. 使用 Hugo 在本地创建博客
在 [Hugo](https://github.com/gohugoio/hugo/releases/tag/v0.111.3) 官网下载最新的版本（选扩展版本）

### 2.1 创建项目
注意项目名称最好和仓库名一致。

{{< admonition tip >}} username 请换成自己的 GitHub 用户名。{{< /admonition >}}
```bash
hugo new site username.github.io
```
### 2.2 安装主题
主题有很多种，选一种你最喜欢的。
```bash
cd username.github.io
git init
git submodule add https://github.com/dillonzq/LoveIt.git themes/LoveIt
```
### 2.3 配置项目
配置参数可以去主题所在的 Github 项目进行了解。以下是 LoveIt 主题的基本配置：
```toml
baseURL = "http://example.org/"

# 更改使用 Hugo 构建网站时使用的默认主题
theme = "LoveIt"

# 网站标题
title = "我的全新 Hugo 网站"

# 网站语言, 仅在这里 CN 大写 ["en", "zh-CN", "fr", "pl", ...]
languageCode = "zh-CN"
# 语言名称 ["English", "简体中文", "Français", "Polski", ...]
languageName = "简体中文"
# 是否包括中日韩文字
hasCJKLanguage = true

# 作者配置
[author]
  name = "xxxx"
  email = ""
  link = ""

# 菜单配置
[menu]
  [[menu.main]]
    weight = 1
    identifier = "posts"
    # 你可以在名称 (允许 HTML 格式) 之前添加其他信息, 例如图标
    pre = ""
    # 你可以在名称 (允许 HTML 格式) 之后添加其他信息, 例如图标
    post = ""
    name = "文章"
    url = "/posts/"
    # 当你将鼠标悬停在此菜单链接上时, 将显示的标题
    title = ""
  [[menu.main]]
    weight = 2
    identifier = "tags"
    pre = ""
    post = ""
    name = "标签"
    url = "/tags/"
    title = ""
  [[menu.main]]
    weight = 3
    identifier = "categories"
    pre = ""
    post = ""
    name = "分类"
    url = "/categories/"
    title = ""

# Hugo 解析文档的配置
[markup]
  # 语法高亮设置 (https://gohugo.io/content-management/syntax-highlighting)
  [markup.highlight]
    # false 是必要的设置 (https://github.com/dillonzq/LoveIt/issues/158)
    noClasses = false
```

### 3. 创建 Github Actions 配置文件
Github Actions 类似 Gitlab 的 CI/CD 功能，可以帮助我们自动构建、测试和部署项目。而且很多具体的 Action 都有专门的 Github 项目维护，我们
的配置页来自于 Hugo 的 [技术文档](https://gohugo.io/hosting-and-deployment/hosting-on-github/) 。

现在项目根目录创建如下目录：`.github/workflows`，再在此目录下创建任意名称，yaml 格式的配置文件。Github 会自动识别并根据此目录下的配置文件
执行设定的 Action 。
```yaml
# Sample workflow for building and deploying a Hugo site to GitHub Pages
name: Deploy Hugo site to Pages

on:
  # Runs on pushes targeting the default branch
  push:
    branches:
      - master

  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:

# Sets permissions of the GITHUB_TOKEN to allow deployment to GitHub Pages
permissions:
  contents: read
  pages: write
  id-token: write

# Allow only one concurrent deployment, skipping runs queued between the run in-progress and latest queued.
# However, do NOT cancel in-progress runs as we want to allow these production deployments to complete.
concurrency:
  group: "pages"
  cancel-in-progress: false

# Default to bash
defaults:
  run:
    shell: bash

jobs:
  # Build job
  build:
    runs-on: ubuntu-latest
    env:
      HUGO_VERSION: 0.111.3
    steps:
      - name: Install Hugo CLI
        run: |
          wget -O ${{ runner.temp }}/hugo.deb https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-amd64.deb \
          && sudo dpkg -i ${{ runner.temp }}/hugo.deb
      - name: Install Dart Sass Embedded
        run: sudo snap install dart-sass-embedded
      - name: Checkout
        uses: actions/checkout@v3
        with:
          submodules: recursive
          fetch-depth: 0
      - name: Setup Pages
        id: pages
        uses: actions/configure-pages@v3
      - name: Install Node.js dependencies
        run: "[[ -f package-lock.json || -f npm-shrinkwrap.json ]] && npm ci || true"
      - name: Build with Hugo
        env:
          # For maximum backward compatibility with Hugo modules
          HUGO_ENVIRONMENT: production
          HUGO_ENV: production
        run: |
          hugo \
            --gc \
            --minify \
            --baseURL "${{ steps.pages.outputs.base_url }}/"
      - name: Upload artifact
        uses: actions/upload-pages-artifact@v1
        with:
          path: ./public

  # Deployment job
  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v2
```

### 4. 将博客与远程仓库建立连接
```bash
git remote add origin git@github.com:username.github.io.git
```

### 5. 写博客，上传即自动部署
```bash
hugo new posts/xxx.md
git add .
git commit -m 'init my blog'
git push
```
最后访问`username.github.io`即可。