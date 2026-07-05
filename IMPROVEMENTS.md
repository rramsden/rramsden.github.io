# Code Review & Improvement Plan

A review of the current Jekyll site (layouts, CSS, config, assets) with a
prioritized plan to modernize. The site works and builds cleanly — this is
about polish, accessibility, performance, and making it pleasant to maintain
as the post count grows.

---

## Summary

| Area | Status | Notes |
|------|--------|-------|
| Build | ✅ Good | Jekyll 4.4, clean build, no warnings |
| Structure | 🟡 OK | `_includes/` exists but is empty; layouts inline everything |
| Accessibility | 🔴 Gaps | No focus styles, no skip link, nav unlabeled |
| Performance | 🔴 Gaps | 600KB–900KB unoptimized PNGs, no lazy-loading, no dimensions |
| CSS | 🟡 Dated | Media-query font swaps instead of `clamp()`; no logical props |
| SEO/meta | 🟡 OK | seo-tag present but under-configured (no author/social/image) |
| Scalability | 🟡 Watch | No pagination; homepage grows unbounded |

---

## Priority 1 — Accessibility (do first, quick wins) ✅ DONE

These are correctness issues, not nice-to-haves.

- [x] **No visible focus styles.** Added `:focus-visible` outlines using `--accent`.
- [x] **No skip-to-content link.** Added a visually-hidden "Skip to content"
  anchor as the first focusable element, targeting `<main id="content">`.
- [x] **Nav is unlabeled.** Added `aria-label="Primary"` to `<nav>`; `<main>`
  now has `id="content"`.
- [x] **Logo-only header.** Confirmed logo `alt` uses the full site title.
- [x] **Declare `color-scheme`.** Added `color-scheme: light;` plus a
  `<meta name="color-scheme">`.

## Priority 2 — Performance

The keyboard post pulls in ~5MB of images. This is the biggest real-world
issue for readers on mobile/slow connections.

- [ ] **Optimize images.** The `mech-keyboard/*.png` files are 500KB–900KB
  each. Re-encode to WebP (and/or compress PNGs). Target < 200KB each.
- [ ] **Lazy-load images.** Markdown `![]()` output has no `loading="lazy"`.
  Options: a small kramdown post-process, an include/figure shortcode, or a
  `_plugins` hook. Add `loading="lazy"` + `decoding="async"`.
- [ ] **Set image dimensions** to prevent layout shift (CLS). Requires either
  authored width/height or an image-processing step.
- [ ] Consider `jekyll-postcss`/asset pipeline or simply minify `style.css`
  for production (`JEKYLL_ENV=production`).

## Priority 3 — Modernize components & CSS ("this stuff is old") ✅ MOSTLY DONE

Refactor toward reusable includes and modern CSS. This is the "update
components" work.

- [x] **Use the empty `_includes/` dir.** Extracted `head.html`, `header.html`,
  `footer.html`, and `post-card.html`. Layouts are now thin.
- [x] **Fluid typography with `clamp()`.** Replaced the `@media (max-width:640px)`
  font-size swap with `clamp(1rem, 0.94rem + 0.3vw, 1.125rem)`.
- [x] **Logical properties.** Swapped `margin-left`/`padding-left` for
  `margin-inline`/`padding-block`/`padding-inline-start` etc.
- [x] **`text-wrap: balance`** on headings and **`text-wrap: pretty`** on body
  paragraphs.
- [x] **Refined typography:** added `hanging-punctuation`.
  (Note: fixed a stray `font-size: 01.45rem` typo in `.post-content`.)
- [ ] **`:where()` low-specificity reset** — not yet done (optional polish).
- [ ] **Optional variable web-font.** The system serif stack is fast and
  looks good, but a self-hosted variable serif (e.g. Source Serif 4, Newsreader,
  or Literata) would give a consistent, distinctive look across all devices.
  Trade-off: ~30–80KB font download. Recommend `font-display: swap` + preload.

## Priority 4 — Config, SEO & metadata

- [ ] **Flesh out `_config.yml` for seo-tag:** add `author` as a structured
  object, `social` (links/handles), a default `image` for Open Graph/Twitter
  cards, and `logo`. Empty `email: ""` should be filled or removed.
- [ ] **Default social share image.** Right now shared links have no preview
  image. Add one (can reuse `logo.svg` rasterized, or a simple OG card).
- [ ] **Favicon fallback.** Only an SVG favicon is set. Add a PNG/ICO fallback
  for browsers/apps that don't support SVG favicons.

## Priority 5 — Scalability & authoring (you plan to write a lot)

- [ ] **Pagination or archive.** The homepage lists *every* post. Add
  `jekyll-paginate-v2` or an `/archive/` page grouped by year, and keep the
  homepage to the latest N.
- [ ] **Tag pages.** Tags render on posts but don't link anywhere. Generate
  `/tags/<tag>/` index pages (plugin or a simple generator) so tags are useful.
- [ ] **Drafts workflow.** Use `_drafts/` + `jekyll serve --drafts` for
  in-progress Notion imports so half-finished posts don't publish.
- [ ] **Reading time / word count** (optional) — a nice touch for readers.
- [ ] **Notion → post script.** Add a `bin/new-post.sh` that ingests a Notion
  markdown export, strips its junk, adds front matter, and relocates images.

## Priority 6 — Quality gates (nice to have)

- [ ] **HTML-Proofer in CI.** Add a step to the Pages workflow that runs
  `htmlproofer ./_site` to catch broken links/images before deploy.
- [ ] **`.editorconfig`** for consistent formatting across editors.
- [ ] **Homepage Liquid nit.** The year-marker is an `<h2>` inside an `<li>`
  that's a sibling of post `<li>`s — slightly odd semantics. Cleaner: render
  a `<section>` per year with its own list, or use `<h2>` outside the `<ul>`.

---

## Suggested order of work

1. **P1 accessibility** (small, high-value) — focus styles, skip link, aria.
2. **P3 component refactor** — move to `_includes/`, adopt `clamp()` +
   logical properties + `text-wrap`. This is the "modernize old stuff" ask.
3. **P2 image performance** — biggest win for real readers.
4. **P4 SEO/meta** + **P5 scalability** as the blog fills up.
5. **P6 quality gates** last.

Nothing here is urgent or breaking — the site is solid. These changes make it
faster, more accessible, and much nicer to maintain as you publish regularly.
