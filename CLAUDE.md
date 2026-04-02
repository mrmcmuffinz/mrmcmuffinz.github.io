# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

A personal Hugo static site (mrmcmuffinz.dev) using the [Blowfish theme](https://blowfish.page/). Content is Markdown with TOML front matter. Deployed to GitHub Pages on push to `main`.

## Common Commands

```bash
# After cloning — initialize the Blowfish theme submodule
git submodule update --init --recursive

# Dev server — full reload, no cache, drafts visible
hugo server --disableFastRender --noHTTPCache -D

# Same but with explicit baseURL (useful in devcontainers)
hugo server --disableFastRender --noHTTPCache -D --baseURL http://localhost:1313/

# Build the site
hugo

# New post (use full path: content/posts/slug.md)
hugo new content/posts/my-post-title.md
```

## CI/CD

- **`hugo-pages.yml`** — builds with Hugo `0.152.2` (extended) and deploys to GitHub Pages on push to `main`. Local Hugo is `0.159.2` — be aware of version drift.
- **`linkedin-rss.yml`** — cron job that posts latest RSS entry to LinkedIn. Tracks the last posted article in `.github/.lastPost.txt` — don't modify that file unless you understand the LinkedIn posting flow.
- The Blowfish theme is a **git submodule** — CI checks it out with `submodules: true`.

## Content

- `content/posts/` — blog posts
- `content/about.md`, `contact.md`, etc. — standalone pages
- Front matter uses TOML (`+++` delimiters), not YAML

### Post Front Matter

```toml
+++
date = '2026-01-01T00:00:00-05:00'
draft = true
title = 'Post Title'
tags = ["tag1", "tag2"]
categories = ["category"]
description = "Short description for SEO/preview."
layout = "post"
+++
```

## Configuration

- `config/_default/hugo.toml` — core settings (base URL, theme, taxonomies)
- `config/_default/params.toml` — Blowfish theme params (color scheme, dark mode, Disqus comments)
- `assets/css/custom.css` — CSS overrides
- `layouts/` — Hugo template overrides (use these instead of modifying `themes/blowfish/` directly)

## Skills

- **blog-post-writer** (`.claude/skills/blog-post-writer/SKILL.md`) — Read
  before writing, drafting, or revising any blog post. Defines voice, structure,
  Hugo conventions, and the multi-phase writing workflow.