# redwolf2019.github.io

个人博客，线上地址 https://redwolf2019.github.io ，仓库 `redwolf2019/redwolf2019.github.io`。

- 主题：Jekyll + [Chirpy](https://github.com/cotes2020/chirpy-starter)（gem `jekyll-theme-chirpy ~> 7.6`），由 chirpy-starter 拷入，未保留其 `.devcontainer`/`.vscode`/`tools`/README/LICENSE。
- 部署：GitHub Actions（`.github/workflows/pages-deploy.yml`），Pages 的 build_type 是 `workflow`，不是分支部署。push 到 `main` 即构建发布，约 1 分钟。
- 本地不装 Ruby/Jekyll，构建只在 Actions 上跑。

## 写文章

用户只写文章，其余不动。新文章放 `_posts/YYYY-MM-DD-slug.md`，URL 为 `/posts/slug/`：

```markdown
---
title: 文章标题
description: 一两句话概括文章讲了什么，70～150 字以内
date: 2026-10-20 20:00:00 +0800
categories: [随笔]
tags: [生活]
---
正文
```

- `categories` 最多两级，`tags` 随意；分类页、标签页、归档、目录都自动生成。
- 不要写 `layout: post`，Chirpy 默认就是。

## 去 AI 味（每篇文章都要做）

文章交给用户 review 之前，必须过 sepia：
- 新写文章用 `sepia:sepia-write`；改已有文章用 `sepia:sepia-refactor`，长文按 `##` 逐节跑；只诊断不改用 `sepia:sepia-review`。
- 只改措辞。front matter、数字、版本号、命令、代码块、链接、报错原文、术语一字不动，改完对照原文核对。
- sepia 没跑完或不可用时，不把文章说成已完成，也不提请发布。

## SEO / GEO（每篇文章都要做）

站点级已配好，不用每次动：jekyll-seo-tag（title/description/canonical/OG/JSON-LD BlogPosting，作者取 `_config.yml` 的 `author`）、`sitemap.xml`、`feed.xml`、`robots.txt`（全放行）、`social_preview_image`（GitHub 头像），以及给 AI 引擎读的 `/llms.txt`（`llms.txt` 模板，自动列出全部文章）。

写或改文章时检查：
- `description` 必填。它同时是搜索结果摘要、社交卡片描述和 `llms.txt` 里的条目；没写就退回正文第一段。
- `title` 直接写主题词，别用「第一篇」「杂记」这种搜不到的标题。
- 正文开头两三句先给结论或要点，AI 引擎摘引时最常取开头。
- 用 `##` / `###` 分节（也驱动右侧目录），一节讲一件事；步骤用有序列表，对比用表格。
- 文件名 slug 用英文或拼音短词，它就是 URL（`/posts/<slug>/`）。
- 配图写 `![说明文字](路径)`，说明文字不留空；需要专属分享图时在 front matter 加 `image: <路径>`。
- 文章互相引用用站内相对链接，构建时 htmlproofer 会检查死链。
- 内容有更新时加 `last_modified_at`（不写也会按 git 提交时间生成 dateModified）。

站外需要站长本人操作、无法在仓库里完成的：Google Search Console、Bing Webmaster、百度搜索资源平台验证站点，把验证码填进 `_config.yml` 的 `webmaster_verifications`，再在后台提交 `https://redwolf2019.github.io/sitemap.xml`。

## 发布与验证

**未经用户明确授权，不 commit、不 push。** push 即上线，所以文章和任何改动都先停在本地：写完告诉用户改了哪些文件，等用户 review 并明确说「提交」「发布」之类的话再执行。一次授权只管这一次，下一篇照样要等。

授权后：

```bash
git add . && git commit -m "docs: <标题>" && git push
id=$(gh run list -L1 --json databaseId --jq '.[0].databaseId'); gh run watch $id --exit-status
curl -s -o /dev/null -w '%{http_code}\n' https://redwolf2019.github.io/posts/<slug>/
```

构建含 `htmlproofer`（只查站内链接），文章里的站内死链会让部署失败。

## 配置

- `_config.yml`：`lang: zh-CN`、`timezone: Asia/Shanghai`、标题、`url`、头像 `avatar`、评论 `comments` 都在这里。
- `_data/contact.yml`：侧边栏图标，只留了 github 和 rss。twitter/email 已删，因为没填会显示成空链接，也不公开邮箱。
- `_tabs/about.md`：「关于」页。
- `CLAUDE.md` 在 `_config.yml` 的 `exclude` 里，不会发布到站点。
