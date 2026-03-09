# Case Knowledge Base

Personal knowledge base for support case summaries and lessons learned.

Built with [Jekyll](https://jekyllrb.com/) and hosted on [GitHub Pages](https://pages.github.com/).

## How to add a new post

1. Copy your Case Summary `.md` file to the `_posts/` folder
2. Rename it to `YYYY-MM-DD-short-title.md` format (e.g., `2026-02-03-dns-resolution-failure.md`)
3. Add YAML front matter at the top of the file (see example below)
4. Commit and push

### Front matter example

```yaml
---
layout: post
title: "DNS 解析 app.powerbi.com 间歇性失败"
date: 2026-02-03
categories: [DNS, Windows-Server]
tags: [dns, zone, cname, powerbi, azurewebsites]
case_number: "2602030030000451"
---
```

## Local preview

```bash
gem install bundler jekyll
bundle install
bundle exec jekyll serve
```

Then open http://localhost:4000
