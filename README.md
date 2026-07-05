# Keep It Simple Stupid

My personal blog — [rramsden.ca](https://rramsden.ca). A minimal, serif,
read-first Jekyll site. Write markdown, `git push`, done.

## Writing a new post

Create a file in `_posts/` named `YYYY-MM-DD-a-short-slug.md`:

```markdown
---
layout: post
title: "Your Title Here"
date: 2026-07-05 09:00:00 +0900
tags: [writing, ideas]
---

Your first paragraph. This also becomes the excerpt on the homepage.

## A heading

More words...
```

That's the whole workflow:

```sh
vim _posts/2026-07-05-my-new-post.md
git add -A && git commit -m "New post: my new post" && git push
```

GitHub Actions builds and deploys automatically on push to `master`.

## Images

Drop images in `assets/img/` and reference them:

```markdown
![alt text](/assets/img/my-picture.png)
```

## Preview locally

```sh
bundle install          # first time only
bundle exec jekyll serve # http://localhost:4000, auto-reloads
```

## Structure

- `_posts/` — your writing (markdown)
- `_layouts/` — page templates (`default`, `post`)
- `assets/css/style.css` — the theme (fonts, colors, spacing)
- `about.md` — the about page
- `index.html` — the homepage post list
- `_config.yml` — site settings

## Theme notes

Tuned for readability: a native serif stack (Iowan / Charter / Palatino /
Georgia), a ~66-character measure, generous line-height, old-style numerals,
and an automatic light/dark warm-paper palette. To tweak, edit the `:root`
variables at the top of `assets/css/style.css`.
