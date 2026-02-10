# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Personal tech blog for Sümer Cip (sumercip.com), built with Hugo. Topics: observability, distributed systems, Go, Python, performance.

## Build & Development

```bash
hugo server          # Local dev server (hot-reload at localhost:1313)
hugo --minify        # Production build (output: ./public)
hugo new posts/slug-name.md   # New post (creates draft from archetypes/default.md)
```

Hugo must be installed (`brew install hugo` on macOS).

## Architecture

- **config.toml** — Site config: base URL, theme selection, markdown/syntax settings, nav links
- **content/posts/** — Blog posts in Markdown with YAML frontmatter (title, date, draft, description, images)
- **content/{about,articles,talks}.md** — Standalone pages (articles/talks link to external content)
- **themes/my_yinyang/** — Customized fork of hugo-theme-yinyang (git submodule pointing to sumerc/hugo-theme-yinyang)
- **static/** — Images, favicons, diagrams referenced in posts
- **layouts/** — Project-level Hugo template overrides (takes precedence over theme templates)

## Theme (my_yinyang)

Git submodule — changes to theme files should be committed in the submodule repo, not this repo.

Key theme files:
- `themes/my_yinyang/layouts/partials/` — head.html, header.html, footer.html, seo.html
- `themes/my_yinyang/assets/css/index.css` — Main stylesheet (fonts: Poppins for headings, Inter for body, Fira Code for code)

Patch files (`my_theme_changes.patch`, `themes/my_yinyang/my_logo.patch`) document past customizations.

## Deployment

GitHub Actions (`.github/workflows/gh-pages.yml`): pushes to `master` trigger `hugo --minify` and deploy to GitHub Pages via `peaceiris/actions-gh-pages@v3`.

## Hugo Config Notes

- Goldmark renderer with `unsafe = true` (raw HTML allowed in Markdown)
- Pygments syntax highlighting with `guessSyntax = true`, tab width 2
- Post frontmatter supports `toc: true` for table of contents and `images` for Twitter cards
