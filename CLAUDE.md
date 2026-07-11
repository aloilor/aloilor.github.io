# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Personal software-engineering blog ("aloilor's archive") built with **Jekyll** and the **Minimal Mistakes** theme, deployed via **GitHub Pages**. The live site is https://aloilor.github.io/ and it builds automatically from the `master` branch — there is no CI build step to run for deployment; pushing to `master` publishes.

Almost all day-to-day work is **authoring Markdown posts**, not editing code. The theme machinery (layouts, includes, sass, Rakefile, JS build) is vendored from the upstream Minimal Mistakes theme and rarely needs to change.

## Local development

Ruby/Bundler + Jekyll (mirrors the GitHub Pages build via the `github-pages` gem):

```bash
bundle install            # first-time setup
bundle exec jekyll serve  # local server with live reload at http://localhost:4000
bundle exec jekyll build  # one-off build into _site/
```

`_config.yml` is **not** reloaded by `jekyll serve` — restart the server after changing it.

## Authoring content

- Posts live in `_posts/` and must be named `YYYY-MM-DD-title-slug.md`. The date in the filename sets the publish date and the URL.
- Permalinks are `/:categories/:title/` (see `_config.yml`), so a post's URL derives from its front matter `categories` and `title`.
- Minimal post front matter used in this repo is just `title` and (optionally) `tags` — see existing posts in `_posts/`. Site-wide post defaults (single layout, author profile, read time, ToC, social share) are set under `defaults:` in `_config.yml`, so individual posts don't repeat them.
- `tags` power the `/tags/` archive (`_pages/tag-archive.md`); posts are also listed by date at `/year-archive/`, which is the sole nav entry defined in `_data/navigation.yml`.
- Markdown is kramdown with GFM input; syntax highlighting is Rouge.

## Architecture notes

- **Theme is remote + vendored.** `_config.yml` sets `remote_theme: mmistakes/minimal-mistakes`, but the repo also contains full local copies of `_layouts/`, `_includes/`, `_sass/`, and `assets/js|css`. Local files override the remote theme. Most of these are unmodified upstream theme files — prefer changing `_config.yml`, `_data/`, or `assets/` overrides over editing them.
- **Site-wide behavior is driven by `_config.yml`**, not code: skin (`minimal_mistakes_skin: dark`), search (Lunr, `search: true`), pagination (5 posts/page), plugins, author/footer profile links, and the post `defaults:` block.
- `_data/navigation.yml` = top nav; `_data/ui-text.yml` = theme UI strings.
- `Rakefile` is **upstream theme tooling** (regenerates copyright headers, minifies JS into `assets/js/main.min.js` via `uglify-js`, builds theme docs). It is not needed to author posts or run the site locally; only relevant if modifying the theme's bundled JavaScript.
- `staticman.yml` and the `comments:` block configure Staticman-based comments, but comments are currently **disabled** (`comments.provider` is unset in `_config.yml`, and post defaults set `comments: false`).
- `.travis.yml` references an Algolia docs-indexing job for the upstream theme and is not part of this site's deployment.
