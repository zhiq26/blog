---
title: "Building a Personal Blog with Hugo, GitHub Pages, and Cloudflare"
date: 2026-10-07T00:00:00+08:00
draft: false
description: "A practical guide to setting up Hugo and PaperMod, deploying with GitHub Actions, connecting a custom domain through Cloudflare DNS, and troubleshooting common issues."
slug: "hugo-github-pages-cloudflare-en"
tags: ["Hugo", "GitHub Pages", "GitHub Actions", "Cloudflare"]
categories: ["Blogging"]
ShowToc: true
TocOpen: true
---

I built this blog with Hugo and the PaperMod theme. The source lives in GitHub, GitHub Actions builds and deploys the site, and GitHub Pages serves the resulting files. Cloudflare manages the DNS records for my domain.

This post walks through the setup, from the first local preview to a working site at `https://zhiq.dev/`. It also covers three issues I encountered along the way: a missing draft post, a custom domain validation error, and a stylesheet returning HTTP 404.

The examples use my repository, `zhiq26/blog`, and domain, `zhiq.dev`. Replace them with your own. Shell commands use Bash; on Windows, you can run them in Git Bash.

## 1. How the pieces fit together

| Component | Role |
| --- | --- |
| Hugo | Builds static pages from Markdown, templates, and assets |
| PaperMod | Provides the blog's theme and layout |
| GitHub repository | Stores posts, configuration, the theme reference, and the workflow |
| GitHub Actions | Builds and deploys the site after a push |
| GitHub Pages | Hosts the generated static files |
| Cloudflare DNS | Points the custom domain to GitHub Pages |

This setup uses GitHub Pages for hosting and Cloudflare for DNS. Cloudflare Pages is a separate hosting product.

I verified the setup in three stages: the local site, the default GitHub Pages URL, and finally the custom domain. This made it easier to identify which part needed attention when something failed.

## 2. Create the Hugo project

Install Git and Hugo, then check that both commands are available:

```bash
git --version
hugo version
```

See the [Hugo installation guide](https://gohugo.io/installation/) for your operating system. Keep a note of your local Hugo version so you can use the same version in CI.

Create a repository named `blog` on GitHub. The following commands assume it does not already contain a Hugo project:

```bash
git clone https://github.com/zhiq26/blog.git
cd blog
hugo new site . --force
git submodule add https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
```

Edit `hugo.toml` in the project root:

```toml
baseURL = "https://zhiq.dev/"
languageCode = "zh-cn"
title = "ZhiQ"
theme = "PaperMod"
```

This is the configuration for my existing site. For an English-only blog, use `languageCode = "en"`. Adding an English post to an existing Chinese blog does not require changing the site's language setting; a full multilingual setup is a separate configuration task.

The `baseURL` above uses the final domain. During deployment, the workflow will override it with the current GitHub Pages URL.

Add a `.gitignore`:

```gitignore
/public/
/resources/_gen/
/.hugo_build.lock
```

`public/` contains generated output. CI rebuilds it, so it does not belong in the source repository.

Create a test post and start the development server:

```bash
hugo new content posts/hello-world.md
hugo server -D
```

Open `http://localhost:1313/`. Check that the homepage and test post appear. The `-D` flag includes drafts in the preview.

Commit the project:

```bash
git add .
git commit -m "Initialize Hugo blog"
git push origin main
```

Because the theme is a Git submodule, clone the repository on another machine with:

```bash
git clone --recurse-submodules https://github.com/zhiq26/blog.git
```

If you have already cloned it without submodules, run:

```bash
git submodule update --init --recursive
```

## 3. Deploy with GitHub Actions

In the repository, open **Settings → Pages → Build and deployment** and set **Source** to **GitHub Actions**.

Leave the custom domain unset for now. First, make the default Pages URL work.

Create `.github/workflows/hugo.yaml`. The example below is a compact workflow for this PaperMod project. It retains the action major versions used during the setup; it is not a list of the latest releases. For a new project, the [official Hugo Pages template](https://gohugo.io/host-and-deploy/host-on-github-pages/) is also a useful starting point.

**Replace `0.150.0` with your local Hugo version.** For example, if `hugo version` reports `v0.152.2+extended`, set `HUGO_VERSION` to `"0.152.2"`. If your theme or customizations require Dart Sass, Node.js, or Hugo Modules, add the corresponding dependency setup.

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
      HUGO_VERSION: "0.150.0" # Replace with your local version
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

The important details are:

- `submodules: recursive` fetches the PaperMod theme.
- `pages: write` and `id-token: write` allow the deployment to authenticate and publish.
- `needs: build` makes deployment wait for the build and artifact upload.
- `--baseURL` generates links for the current Pages URL, overriding the value in `hugo.toml`.

The timezone reflects my own setup. Choose the timezone appropriate for your publishing workflow.

Commit and push:

```bash
git add .github/workflows/hugo.yaml
git commit -m "Add GitHub Pages deployment workflow"
git push
```

Open the repository's **Actions** tab. Once both `build` and `deploy` succeed, visit:

```text
https://zhiq26.github.io/blog/
```

This is a project site, so its default URL includes the repository name, `/blog/`. A custom domain will later serve it from the domain's root.

## 4. Why did my test post disappear?

After the first successful deployment, the homepage loaded but Hello World was missing. That was expected: the post was still a draft.

New posts commonly start with this front matter:

```yaml
draft: true
```

My local preview used `hugo server -D`, which includes drafts. The production build in the workflow did not include `-D`, so it excluded the post.

When you are ready to publish, change the front matter:

```yaml
---
title: "Hello World"
date: 2026-10-06T22:00:00+08:00
draft: false
---
```

Then preview without the draft flag:

```bash
hugo server
```

If the post still does not appear, check whether `date` or `publishDate` is in the future and whether the file has been committed to the deployment branch.

## 5. Connect the custom domain

Before adding records, make sure Cloudflare is actually authoritative for the domain: the zone should be **Active**, and the registrar should use the nameservers assigned by Cloudflare.

In the repository, open **Settings → Pages → Custom domain**, enter `zhiq.dev`, and save.

Then open **DNS → Records** in Cloudflare and add:

| Type | Name | Target | Proxy status |
| --- | --- | --- | --- |
| A | @ | 185.199.108.153 | DNS only |
| A | @ | 185.199.109.153 | DNS only |
| A | @ | 185.199.110.153 | DNS only |
| A | @ | 185.199.111.153 | DNS only |
| CNAME | www | zhiq26.github.io | DNS only |

`@` represents the apex domain, `zhiq.dev`. Leave TTL at **Auto** if you do not need a specific value.

The `www` target is `zhiq26.github.io`, without a protocol or repository path. Resolve any conflicting records for the same host, especially A, AAAA, or CNAME records pointing to another service.

Use **DNS only**, shown as a gray cloud, during the initial setup. With **Proxied**, Cloudflare returns its proxy addresses, which can interfere with GitHub's DNS validation.

For this Actions-based deployment, the domain is configured in the Pages settings. A repository `CNAME` file is not required to establish that binding.

## 6. Troubleshoot domain validation

During setup, GitHub reported:

```text
Both zhiq.dev and its alternate name are improperly configured
Domain does not resolve to the GitHub Pages server.
(NotServedByPagesError)
```

All five records were initially proxied. After I switched them to DNS only, local DNS queries returned the correct Pages addresses, but GitHub still displayed the error temporarily.

The message alone does not identify the exact cause. Work through these checks:

1. Confirm that both the apex and `www` records are DNS only.
2. Check for conflicting records pointing elsewhere.
3. Query both hostnames, comparing recursive resolvers if needed.
4. Once the results are correct, allow caches to expire and retry the Pages check.

On Windows, you can use:

```powershell
nslookup zhiq.dev
nslookup www.zhiq.dev
nslookup zhiq.dev 1.1.1.1
nslookup www.zhiq.dev 8.8.8.8
```

The apex should resolve to the Pages IPv4 addresses in the table. The `www` hostname should resolve through a CNAME to `zhiq26.github.io`.

DNS changes can take up to 24 hours to propagate. A correct answer from your local resolver does not mean every other resolver has refreshed its cache.

### Optional: verify domain ownership

Under your personal GitHub account's **Settings → Pages → Add a domain**, you can verify ownership of the domain.

GitHub supplies a TXT record name and verification value. Add that record in Cloudflare, then return to GitHub to verify it. For my account, the record name has this form:

```text
_github-pages-challenge-zhiq26.zhiq.dev
```

Use the exact name and value GitHub provides, and keep the TXT record after verification.

Ownership verification protects the domain's association with your account. It does not replace the A/CNAME records and is not a required fix for `NotServedByPagesError`.

## 7. Enable HTTPS

Once DNS and domain validation are working, wait for GitHub to provision the certificate. Then enable **Enforce HTTPS** in the Pages settings.

Check that:

- `https://zhiq.dev/` loads successfully.
- `https://www.zhiq.dev/` redirects to the apex domain.

The `.dev` top-level domain is HSTS-preloaded, so browsers require HTTPS. Certificate provisioning is therefore part of getting the site online.

If the HTTPS option is unavailable, check the DNS and certificate status. If you have CAA records, check whether they restrict certificate issuance. Avoid repeatedly changing A records that already resolve correctly.

## 8. Fix a stylesheet returning 404

After the domain became reachable, I encountered another error:

```text
stylesheet.<hash>.css
Failed to load resource: the server responded with a status of 404
```

The site had moved from `https://zhiq26.github.io/blog/` to `https://zhiq.dev/`. An old build could still contain resource URLs with the `/blog/` prefix.

A filename and status code are not enough to confirm that diagnosis. Open the browser's developer tools, select **Network**, and inspect the stylesheet's full **Request URL**.

| Request URL | What to check |
| --- | --- |
| `https://zhiq.dev/blog/assets/css/...` | An old base URL or an outdated deployment artifact |
| `https://zhiq.dev/assets/css/...` | A missing output file or cached HTML referencing an old hash |

Confirm the final URL in `hugo.toml`:

```toml
baseURL = "https://zhiq.dev/"
```

Keep the workflow's build argument:

```bash
--baseURL "${{ steps.pages.outputs.base_url }}/"
```

After changing the custom domain, run the full workflow again, including both build and deploy. You can select **Run workflow** in Actions, push a configuration change, or trigger it with an empty commit:

```bash
git commit --allow-empty -m "Rebuild site for custom domain"
git push
```

Once deployment finishes, hard-refresh the page with `Ctrl + Shift + R` on Windows or Linux, or `Command + Shift + R` on macOS.

For further investigation, build locally:

```bash
hugo --gc --minify --baseURL "https://zhiq.dev/"
```

Inspect the stylesheet URL in `public/index.html` and check that the referenced file exists under `public/assets/css/`. Fix the source configuration and rebuild instead of editing generated HTML.

## 9. The everyday publishing workflow

With the infrastructure in place, publishing becomes straightforward: write Markdown, preview locally, commit, push, and let Actions deploy.

To keep a post and its images together, use a Hugo leaf bundle. For example, put the article at `content/posts/my-first-post/index.md` and an image at `content/posts/my-first-post/screenshot.png`.

Reference the image from the article:

```markdown
![A screenshot of the page](screenshot.png)
```

This English tutorial can be saved as `content/posts/hugo-github-pages-cloudflare-en.md`. Its distinct slug allows it to coexist with the Chinese article in a single-language site. If you later configure Hugo's multilingual mode, use that configuration's language-specific paths or filename conventions instead.

While drafting:

```bash
hugo server -D
```

Before publishing, set `draft: false`, check that the publication date is not in the future, and preview without `-D`:

```bash
hugo server
```

Once the page and images look right:

```bash
git add content/
git commit -m "Publish English Hugo setup tutorial"
git push
```

My site is now accessible at `zhiq.dev`. From here, I can gradually add an About page, archives, search, and RSS while spending more time writing.

## References

Tool and action versions change. Check the documentation against the versions you use when following this guide.

- [Hugo installation](https://gohugo.io/installation/)
- [Hosting Hugo on GitHub Pages](https://gohugo.io/host-and-deploy/host-on-github-pages/)
- [Hugo front matter](https://gohugo.io/content-management/front-matter/)
- [Hugo page bundles](https://gohugo.io/content-management/page-bundles/)
- [PaperMod](https://github.com/adityatelange/hugo-PaperMod)
- [Managing a custom domain for GitHub Pages](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/managing-a-custom-domain-for-your-github-pages-site)
- [Verifying a custom domain for GitHub Pages](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/verifying-your-custom-domain-for-github-pages)
- [Securing a GitHub Pages site with HTTPS](https://docs.github.com/en/pages/getting-started-with-github-pages/securing-your-github-pages-site-with-https)
- [Cloudflare DNS proxy status](https://developers.cloudflare.com/dns/proxy-status/)
