# Fesq Journal — blog.fesq.in

Journal for Fesq — blog.fesq.in. Built with Astro, deployed to GitHub Pages.

## Local development

```sh
bun install
bun run dev
```

Open http://localhost:4321.

## Build

```sh
bun run build
```

Output lands in `dist/`.

## Deploy

Push to `main` triggers the GitHub Actions workflow that builds and deploys to GitHub Pages.

## How to post an article

Add a new markdown file to `src/content/blog/`:

```markdown
---
title: Post title
description: One-line summary for cards and RSS.
pubDate: 2026-10-24
tags: [example]
draft: true
---
```

Write with `draft: true` first, verify locally, then set it to `false` — commit + push to publish.
