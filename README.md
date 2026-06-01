# Kilian's Group Blog

Source for **<https://kilian-group.github.io/blog/>** — the Weinberger lab's research blog at Cornell.

## Run the site locally

```bash
docker compose up
```

Open <http://127.0.0.1:8080/>. Edits to markdown, `_layouts/`, `_includes/`, and `_data/` hot-reload. Changes to `_config.yml` require `docker compose restart jekyll`. Stop with `docker compose down`.

Built on [al-folio v1.x](https://github.com/alshedivat/al-folio). For theme features, plugin catalog, and broader customization, see [`README_template.md`](README_template.md).

---

## Writing a new post

### 1. Create the file

`_posts/YYYY-MM-DD-your-slug.md`. The filename date sets the publish date (and the post URL's year) unless overridden by `date:` in frontmatter.

> **Future-dated posts are hidden** unless you set `future: true` in `_config.yml`. To preview a draft with a future `date:` locally, flip that on; revert before pushing if you want production to honor the publish date.

### 2. Frontmatter

Pick a layout:

- **`layout: post`** — plain markdown. Good for short notes.
- **`layout: distill`** — [distill.pub](https://distill.pub)-style with author block, sidebar TOC, inline `<d-cite>` citations, and an auto-generated references appendix. Recommended for technical posts.

Distill template (copy/adapt):

```yaml
---
layout: distill
title: "Your post title"
date: 2026-06-01
description: "One-line summary used in social previews and post listings."
tags: tag-one tag-two
categories: research
featured: false           # set true to highlight at the top of the index
giscus_comments: false
related_posts: false

authors:
  - name: First Last
    url: "https://your-site.com/"
    affiliations:
      name: Cornell University

bibliography: YYYY-MM-DD-your-slug.bib   # optional; see step 4

toc:
  - name: Section One
  - name: Section Two
    subsections:
      - name: Subsection
---
```

A worked example lives at [`_posts/2026-05-28-nvidia-dgx-station.md`](_posts/2026-05-28-nvidia-dgx-station.md).

### 3. Embed figures

Drop images into `assets/img/posts/<slug>/`, then embed with the al-folio include:

```liquid
{% include figure.liquid loading="eager" path="assets/img/posts/<slug>/fig.png" class="img-fluid rounded z-depth-1" %}
<div class="caption">Caption text here.</div>
```

Side-by-side figures: wrap two `figure.liquid` includes in a Bootstrap `<div class="row">` (see the worked example).

### 4. Citations (distill only)

Put your BibTeX at **`assets/bibliography/<slug>.bib`** (not `_bibliography/` — that path is for the disabled jekyll-scholar publications page and is loaded differently).

Reference the file in frontmatter:

```yaml
bibliography: YYYY-MM-DD-your-slug.bib
```

Cite inline:

```html
...as shown in our paper <d-cite key="entrykey"></d-cite>.
```

The references list is appended automatically. Do **not** write a manual `## References` section.

### 5. Preview, then ship

```bash
docker compose up       # http://127.0.0.1:8080/
```

When happy, `git add` + `git commit` + `git push origin main`. CI deploys within a couple of minutes — see [Deploying](#deploying).

---

## Deploying

Push to `main`. [`.github/workflows/deploy.yml`](.github/workflows/deploy.yml) builds with `JEKYLL_ENV=production` and publishes `_site/` to the `gh-pages` branch via `JamesIves/github-pages-deploy-action`. Live within ~2–3 minutes of workflow completion.

Notes:

- The workflow rewrites `giscus.repo` in `_config.yml` to the current GitHub repository at build time. Leave that field empty in source.
- `baseurl:` in `_config.yml` is currently empty, so production URLs resolve directly under `kilian-group.github.io/blog/` (the `/blog/` prefix comes from the repo name, since this is a project page). If you ever move to a user/org page or change the project name, update `baseurl` to match.
- Post permalinks are `/:year/:title/`.

---

## Repo layout

| Path | Purpose |
| --- | --- |
| `_posts/` | Blog posts (one markdown file per post). |
| `_pages/about.md` | Landing page (renders the post feed at `/`). |
| `_pages/404.md` | 404 page. |
| `assets/bibliography/<slug>.bib` | BibTeX referenced by distill posts (per-post). |
| `assets/img/posts/<slug>/` | Per-post image assets. |
| `assets/css/main.scss` | Local SCSS — Fira Sans, Fira Code, Cornell red `#b31b1b`. |
| `_bibliography/papers.bib` | Used by jekyll-scholar for the (currently disabled) publications page. |
| `_config.yml` | Site identity, URL config, feature toggles, plugin list. |
| `.github/workflows/deploy.yml` | CI build + deploy to `gh-pages`. |
| `README_template.md` | Upstream al-folio theme reference. |
