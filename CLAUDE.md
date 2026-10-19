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
date: 2026-10-20 20:00:00 +0800
categories: [随笔]
tags: [生活]
---
正文
```

- `categories` 最多两级，`tags` 随意；分类页、标签页、归档、目录都自动生成。
- 不要写 `layout: post`，Chirpy 默认就是。

## 发布与验证

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
