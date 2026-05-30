# scytacki.github.io

Source for <https://scytacki.github.io> — reference notes on web development,
accessibility, and related topics.

Suggestions and PRs welcome.

## Adding a note

Create a new `.md` file at the repo root with Jekyll frontmatter:

```yaml
---
layout: page
title: "Your title"
description: "One-line summary used for SEO and the link card."
permalink: /your-slug/
---
```

Then add a bullet linking to it from `index.md`.

## Local preview (optional)

GitHub Pages builds the site automatically on push. For local preview:

```bash
bundle init
echo 'gem "github-pages", group: :jekyll_plugins' >> Gemfile
bundle install
bundle exec jekyll serve
```
