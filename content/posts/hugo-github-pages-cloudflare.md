---
title: "用 Hugo、GitHub Pages 和 Cloudflare 搭建个人博客"
date: 2026-10-06T23:00:00+08:00
draft: false
description: "记录 zhiq.dev 的搭建过程：Hugo 本地预览、GitHub Actions 自动部署、Cloudflare DNS、自定义域名与常见问题排查。"
slug: "hugo-github-pages-cloudflare"
tags: ["Hugo", "GitHub Pages", "GitHub Actions", "Cloudflare"]
categories: ["博客搭建"]
ShowToc: true
TocOpen: true
---

我用 Hugo 和 PaperMod 搭建了这个博客，将源码放在 GitHub，通过 GitHub Actions 自动构建并发布到 GitHub Pages，最后用 Cloudflare 管理域名解析。

这篇文章记录从本地预览到通过 `https://zhiq.dev/` 访问的完整流程，也整理了搭建时遇到的三个问题：草稿在线上不显示、域名校验报错，以及切换域名后 CSS 返回 404。

本文的示例仓库是 `zhiq26/blog`，域名是 `zhiq.dev`。复用时请替换为自己的用户名、仓库和域名。示例命令使用 Bash；Windows 用户可以使用 Git Bash。

## 1. 先弄清楚各部分负责什么

| 组件 | 作用 |
| --- | --- |
| Hugo | 将 Markdown 和主题编译成静态网页 |
| PaperMod | 提供博客的页面样式和布局 |
| GitHub 仓库 | 保存文章、配置、主题引用和工作流 |
| GitHub Actions | 在推送源码后执行构建和部署 |
| GitHub Pages | 托管构建生成的静态文件 |
| Cloudflare DNS | 将自己的域名解析到 GitHub Pages |

这里使用的是 **GitHub Pages 托管网站**，Cloudflare 负责 DNS。并没有使用 Cloudflare Pages。

我按三个阶段验证：先让本地页面正常，再让 GitHub Pages 默认地址正常，最后绑定自定义域名。每次只增加一层配置，排错比较容易。

## 2. 创建 Hugo 项目并添加主题

先安装 Git 和 Hugo，并检查命令是否可用：

```bash
git --version
hugo version
```

Hugo 安装方法参考[官方安装说明](https://gohugo.io/installation/)。记录本地 Hugo 版本，后续 CI 尽量使用同一个版本。

在 GitHub 创建仓库 `blog`。以下命令假设仓库尚未包含 Hugo 项目，已有项目不要重复初始化：

```bash
git clone https://github.com/zhiq26/blog.git
cd blog
hugo new site . --force
git submodule add https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
```

编辑根目录的 `hugo.toml`：

```toml
baseURL = "https://zhiq.dev/"
languageCode = "zh-cn"
title = "ZhiQ"
theme = "PaperMod"
```

这里先写最终域名。部署默认 Pages 地址时，工作流会通过命令行覆盖 `baseURL`。

添加 `.gitignore`：

```gitignore
/public/
/resources/_gen/
/.hugo_build.lock
```

`public/` 是构建产物，每次部署都会重新生成，不需要提交到源码仓库。

创建测试文章：

```bash
hugo new content posts/hello-world.md
hugo server -D
```

打开 `http://localhost:1313/`，确认首页和测试文章能显示。`-D` 表示预览时包含草稿。

随后提交：

```bash
git add .
git commit -m "Initialize Hugo blog"
git push origin main
```

主题作为 Git submodule 保存。换电脑时，需要同时获取主题：

```bash
git clone --recurse-submodules https://github.com/zhiq26/blog.git
```

如果已经普通克隆了仓库，再执行：

```bash
git submodule update --init --recursive
```

## 3. 开启 GitHub Pages 自动部署

进入仓库的 **Settings → Pages → Build and deployment**，将 **Source** 设为 **GitHub Actions**。

此时先不填写 Custom domain，先验证默认地址。

在项目中创建 `.github/workflows/hugo.yaml`。下面是适用于本次简单 PaperMod 项目的工作流示例，保留搭建时采用的 Actions 主版本；它不是“永远最新”的版本清单。新项目也可以从 [Hugo 官方 Pages 模板](https://gohugo.io/host-and-deploy/host-on-github-pages/)开始。

**示例中的 Hugo 版本 `0.150.0` 必须按自己的环境调整。** 例如本地输出 `v0.152.2+extended`，这里就填写 `0.152.2`。如果后续主题引入 Sass、Node.js 或 Hugo Modules，还需要补上对应依赖安装步骤。

```yaml
name: Deploy Hugo site to Pages

on:
  push:
    branches:
      - main
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: pages
  cancel-in-progress: false

defaults:
  run:
    shell: bash

jobs:
  build:
    runs-on: ubuntu-latest
    env:
      HUGO_VERSION: "0.150.0" # 替换为与本地一致的版本
      TZ: Asia/Shanghai
    steps:
      - name: Checkout
        uses: actions/checkout@v6
        with:
          submodules: recursive
          fetch-depth: 0

      - name: Install Hugo
        run: |
          curl --fail --location \
            --output "${{ runner.temp }}/hugo.deb" \
            "https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-amd64.deb"
          sudo dpkg -i "${{ runner.temp }}/hugo.deb"
          hugo version

      - name: Setup Pages
        id: pages
        uses: actions/configure-pages@v5

      - name: Build with Hugo
        env:
          HUGO_ENVIRONMENT: production
          HUGO_ENV: production
        run: |
          hugo \
            --gc \
            --minify \
            --baseURL "${{ steps.pages.outputs.base_url }}/"

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v4
        with:
          path: ./public

  deploy:
    runs-on: ubuntu-latest
    needs: build
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

几个配置决定了这个流程能否正确工作：

- `submodules: recursive`：拉取 PaperMod 主题。
- `pages: write` 和 `id-token: write`：授权 Pages 部署。
- `needs: build`：构建成功并上传产物后，才执行部署。
- `--baseURL`：按照 Pages 当前地址生成链接，覆盖本地配置中的值。

提交工作流：

```bash
git add .github/workflows/hugo.yaml
git commit -m "Add GitHub Pages deployment workflow"
git push
```

在仓库 **Actions** 中观察 `build` 和 `deploy`。两者成功后，访问：

```text
https://zhiq26.github.io/blog/
```

仓库名是 `blog`，因此它是项目站点，默认地址包含 `/blog/`。绑定自定义域名后，网站可以从域名根路径访问。

## 4. 为什么本地有文章，线上却没有？

我第一次部署成功后，首页可以打开，但没有 Hello World。这其实符合预期：测试文章仍然是草稿。

Hugo 新文章通常包含：

```yaml
draft: true
```

本地用了 `hugo server -D`，会显示草稿；工作流中的生产构建没有 `-D`，所以不会发布它。

准备发布时，把 front matter 改为：

```yaml
---
title: "Hello World"
date: 2026-10-06T22:00:00+08:00
draft: false
---
```

发布前运行：

```bash
hugo server
```

确认不带 `-D` 也能看到文章，再提交。如果仍然不显示，还要检查 `date` 或 `publishDate` 是否在未来，以及文章是否已提交到部署分支。

## 5. 使用 Cloudflare DNS 绑定自定义域名

前提是域名已经由 Cloudflare 管理 DNS：Cloudflare 区域处于 Active 状态，注册商的 nameserver 已按 Cloudflare 的要求设置。仅在一个尚未生效的 DNS 区域里添加记录，并不会改变公网解析。

先在仓库 **Settings → Pages → Custom domain** 填写：

```text
zhiq.dev
```

保存后，在 Cloudflare 的 **DNS → Records** 添加以下记录：

| 类型 | 名称 | 目标 | Proxy status |
| --- | --- | --- | --- |
| A | @ | 185.199.108.153 | DNS only |
| A | @ | 185.199.109.153 | DNS only |
| A | @ | 185.199.110.153 | DNS only |
| A | @ | 185.199.111.153 | DNS only |
| CNAME | www | zhiq26.github.io | DNS only |

`@` 表示根域名。TTL 可以保留 Auto。

`www` 的目标是用户名对应的 `zhiq26.github.io`，不包含 `https://`，也不包含 `/blog/`。已有同名冲突记录时，需要先处理冲突，特别是指向其他服务的 A、AAAA 或 CNAME。

初次配置使用 **DNS only（灰云）**，让域名直接解析到 GitHub Pages。Proxied（橙云）会返回 Cloudflare 的代理地址，可能影响 GitHub 的 DNS 校验。

本项目通过 Actions 发布，域名绑定在 Pages 设置中管理，不需要靠仓库中的 `CNAME` 文件来设置域名。

## 6. 排查 NotServedByPagesError

我曾遇到：

```text
Both zhiq.dev and its alternate name are improperly configured
Domain does not resolve to the GitHub Pages server.
(NotServedByPagesError)
```

最初五条记录都是 Proxied。切成 DNS only 后，本机已能解析到正确的 Pages IP，但 GitHub 页面仍暂时显示错误。因此，不能只凭这个报错就断言是某一条记录配置错误。

按下面顺序检查：

1. 确认根域名和 `www` 都是 DNS only。
2. 确认没有指向其他服务的冲突记录。
3. 分别查询根域名和 `www`，必要时比较不同递归 DNS 的结果。
4. DNS 正确后，给缓存传播留出时间，再回 Pages 页面重试检查。

Windows 可以执行：

```powershell
nslookup zhiq.dev
nslookup www.zhiq.dev
nslookup zhiq.dev 1.1.1.1
nslookup www.zhiq.dev 8.8.8.8
```

根域名应返回上表中的 Pages IPv4 地址，`www` 应通过 CNAME 指向 `zhiq26.github.io`。DNS 更新可能需要最多 24 小时传播，本机正确不代表每个解析器的缓存都已更新。

### 可选：验证域名所有权

GitHub 个人账户 **Settings → Pages → Add a domain** 可以验证域名所有权。按照页面给出的名称和值，在 Cloudflare 添加 TXT 记录，再返回 GitHub 验证。

本账号对应的记录名称通常形如：

```text
_github-pages-challenge-zhiq26.zhiq.dev
```

验证值必须使用 GitHub 实际提供的内容，验证后保留 TXT 记录。

这一步用于保护域名归属，**不是修复 NotServedByPagesError 的必要步骤**，也不能代替正确的 A/CNAME 配置。

## 7. 完成 HTTPS

域名解析和 Pages 校验正常后，等待 GitHub 签发证书，再在 Pages 设置中勾选 **Enforce HTTPS**。

测试：

- `https://zhiq.dev/` 能正常访问。
- `https://www.zhiq.dev/` 能正常跳转到主域名。

`.dev` 域名启用了 HSTS 预加载，浏览器会要求 HTTPS，所以证书配置是正式上线的一部分。

如果 HTTPS 选项暂时不可用，先检查 DNS、证书状态，以及是否存在限制证书签发的 CAA 记录，避免反复修改已经正确的 A 记录。

## 8. 切换域名后 CSS 404 怎么办？

域名能访问后，我又遇到了样式表加载失败：

```text
stylesheet.<hash>.css
Failed to load resource: the server responded with a status of 404
```

站点原先位于 `https://zhiq26.github.io/blog/`，现在位于 `https://zhiq.dev/`。如果仍使用旧构建，HTML 中可能残留 `/blog/` 资源路径。

但仅凭文件名和 404 还不能确定原因。先在浏览器 DevTools 的 **Network** 中查看 CSS 的完整 Request URL。

| 请求路径 | 排查方向 |
| --- | --- |
| `https://zhiq.dev/blog/assets/css/...` | 检查旧 baseURL 和部署产物 |
| `https://zhiq.dev/assets/css/...` | 检查文件是否在产物中、HTML 是否缓存了旧 hash |

先确认 `hugo.toml`：

```toml
baseURL = "https://zhiq.dev/"
```

工作流仍保留：

```bash
--baseURL "${{ steps.pages.outputs.base_url }}/"
```

绑定域名后，需要重新执行包含 **build 和 deploy** 的完整工作流，让生成的链接跟随新地址。可以在 Actions 中手动 Run workflow，也可以提交实际修改；没有源码变更时，可以用空提交触发：

```bash
git commit --allow-empty -m "Rebuild site for custom domain"
git push
```

等部署完成，再用 `Ctrl + Shift + R` 强制刷新页面。

需要进一步定位时，本地构建：

```bash
hugo --gc --minify --baseURL "https://zhiq.dev/"
```

检查 `public/index.html` 中样式表的地址，并确认对应文件存在于 `public/assets/css/`。不要手工改生成的 HTML，应该修正配置后重新构建。

## 9. 以后如何写文章和发布？

日常流程只需要：写 Markdown、本地预览、提交、推送，等待 Actions 自动部署。

为了让文章与图片放在一起，可以使用 Hugo 的 leaf bundle：

```text
content/posts/my-first-post/
├── index.md
└── screenshot.png
```

在 `index.md` 中引用同目录图片：

```markdown
![页面截图](screenshot.png)
```

这篇教程也可以放在 `content/posts/hugo-github-pages-cloudflare/index.md`，或者直接保存为 `content/posts/hugo-github-pages-cloudflare.md`。

写草稿时：

```bash
hugo server -D
```

发布时，将 `draft` 设为 `false`，确认文章日期不是未来时间，再用不带 `-D` 的预览检查：

```bash
hugo server
```

确认页面和图片正常后：

```bash
git add content/
git commit -m "Publish Hugo blog setup tutorial"
git push
```

现在，`zhiq.dev` 已经能成功访问。后续可以逐步完善 About、Archives、Search 和 RSS，把主要精力放到内容上。

## 参考资料

下面的文档用于核对配置；工具和 Actions 版本会变化，复用教程时请结合自己的版本检查。

- [Hugo：安装](https://gohugo.io/installation/)
- [Hugo：部署到 GitHub Pages](https://gohugo.io/host-and-deploy/host-on-github-pages/)
- [Hugo：Front matter](https://gohugo.io/content-management/front-matter/)
- [Hugo：Page bundles](https://gohugo.io/content-management/page-bundles/)
- [PaperMod 项目](https://github.com/adityatelange/hugo-PaperMod)
- [GitHub：管理 Pages 自定义域名](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/managing-a-custom-domain-for-your-github-pages-site)
- [GitHub：验证自定义域名](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/verifying-your-custom-domain-for-github-pages)
- [GitHub：为 Pages 配置 HTTPS](https://docs.github.com/en/pages/getting-started-with-github-pages/securing-your-github-pages-site-with-https)
- [Cloudflare：DNS 代理状态](https://developers.cloudflare.com/dns/proxy-status/)
