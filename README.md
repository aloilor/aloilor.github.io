# aloilor's archive

A personal software-engineering blog where I document my journey — LeetCode write-ups (mostly hards), system design notes, and the technical details of side projects that tend to spiral into chaos.

The site is built with [Jekyll](https://jekyllrb.com/) and the [Minimal Mistakes](https://mmistakes.github.io/minimal-mistakes/) theme, and is published with [GitHub Pages](https://pages.github.com/).

**Live site:** https://aloilor.github.io/

## Overview

This repository is the source for the blog. Day-to-day work is almost entirely **authoring Markdown posts** — the theme machinery is vendored from Minimal Mistakes and rarely needs to change.

Content so far spans a full side-project build-out: CI/CD from scratch, AWS access setup, infrastructure as code with Terraform, container orchestration, hosting backend APIs, SSL renewal, and a manga scraper with alerting.

Key features provided by the theme and configuration:

- Dark skin, responsive layout, and reading-time estimates
- Client-side search across posts ([Lunr](https://lunrjs.com/))
- Archives by [year](https://aloilor.github.io/year-archive/) and by [tag](https://aloilor.github.io/tags/)
- Table of contents, social share links, and an author profile sidebar
- Atom feed and auto-generated sitemap

## Prerequisites

- [Ruby](https://www.ruby-lang.org/) and [Bundler](https://bundler.io/) — for the local Jekyll build
- [Node.js](https://nodejs.org/) — only if you touch the theme's bundled JavaScript (see [Theme tooling](#theme-tooling))

## Getting started

Clone the repo and install the Ruby dependencies:

```bash
bundle install
```

Run a local server with live reload at http://localhost:4000:

```bash
bundle exec jekyll serve
```

Or produce a one-off build into `_site/`:

```bash
bundle exec jekyll build
```

> [!NOTE]
> `_config.yml` is **not** reloaded by `jekyll serve`. Restart the server after changing it.

The local build uses the [`github-pages`](https://github.com/github/pages-gem) gem, so it mirrors what GitHub Pages runs in production.

## Writing a post

Posts live in `_posts/` and must be named `YYYY-MM-DD-title-slug.md`. The date in the filename sets the publish date and feeds into the URL.

Front matter is intentionally minimal — usually just a title and optional tags:

```markdown
---
title: "My new post"
tags:
  - aws
  - terraform
---

Your content here, written in kramdown-flavored Markdown (GFM input).
```

Site-wide post defaults (single layout, author profile, read time, ToC, social share) are set under `defaults:` in `_config.yml`, so individual posts don't repeat them.

Permalinks follow `/:categories/:title/`, so a post's URL derives from its front matter. Tags automatically populate the `/tags/` archive.

## Project structure

```
.
├── _config.yml         # Site-wide settings: skin, search, pagination, defaults, author profile
├── _posts/             # Blog posts (YYYY-MM-DD-title-slug.md)
├── _pages/             # Standalone pages: year archive, tag archive
├── _data/
│   ├── navigation.yml  # Top navigation
│   └── ui-text.yml     # Theme UI strings
├── _layouts/           # Vendored theme layouts
├── _includes/          # Vendored theme partials
├── _sass/              # Vendored theme styles
├── assets/             # CSS, JS, and images
├── index.html          # Home layout entry point
└── Gemfile             # Ruby dependencies
```

> [!NOTE]
> `_layouts/`, `_includes/`, `_sass/`, and `assets/js|css` are local copies of the upstream Minimal Mistakes theme and override the remote theme. They are mostly unmodified — prefer changes in `_config.yml`, `_data/`, or `assets/` overrides over editing them.

## Deployment

There is no separate build step to run. Pushing to the `master` branch triggers the GitHub Pages build and publishes the site automatically.

## Configuration

Most site behavior is driven by `_config.yml` rather than code:

| Setting | Where | Purpose |
| --- | --- | --- |
| Skin / theme | `minimal_mistakes_skin`, `remote_theme` | Visual appearance |
| Search | `search`, Lunr | Client-side full-text search |
| Pagination | `paginate` | Posts per page (5) |
| Navigation | `_data/navigation.yml` | Top nav links |
| Post defaults | `defaults:` | Layout, ToC, share, read time |
| Author / footer | `author`, `footer` | Profile and social links |

## Theme tooling

The `Rakefile` and `package.json` are upstream Minimal Mistakes tooling — they regenerate copyright headers and minify the theme's JavaScript into `assets/js/main.min.js` via [`uglify-js`](https://www.npmjs.com/package/uglify-js). They are **not** needed to author posts or run the site locally, and are only relevant when modifying the theme's bundled JavaScript.

## Acknowledgements

- [Jekyll](https://jekyllrb.com/) — static site generator
- [Minimal Mistakes](https://mmistakes.github.io/minimal-mistakes/) by Michael Rose — the theme
- [GitHub Pages](https://pages.github.com/) — hosting
