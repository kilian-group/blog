# Kilian's Group Blog

Source for **<https://kilian-group.github.io/blog/>** — the Weinberger lab's research blog at Cornell.

Built on [al-folio v1.x](https://github.com/alshedivat/al-folio). For theme features, plugin catalog, and customization options, see [`README_template.md`](README_template.md).

---

## Writing a new post

1. **Create the post file** at `_posts/YYYY-MM-DD-slug.md`. The filename date determines ordering when frontmatter `date:` is omitted; keep it in sync with `date:` to avoid confusion.

2. **Pick a layout.** Two options:
   - `layout: post` — plain markdown. Good for short notes.
   - `layout: distill` — distill.pub-style with an authors block, sidebar TOC, and inline `<d-cite>` citations. Recommended for technical posts. See [`_posts/2026-05-28-nvidia-dgx-station.md`](_posts/2026-05-28-nvidia-dgx-station.md) for a worked example.

3. **Frontmatter template** (distill):

   ```yaml
   ---
   layout: distill
   title: "Your post title"
   date: 2026-06-01
   description: "One-line summary used in social previews and post listings."
   tags: tag-one tag-two
   categories: research
   featured: false
   giscus_comments: false
   related_posts: false

   authors:
     - name: First Last
       url: "https://your-site.com/"
       affiliations:
         name: Cornell University

   bibliography: YYYY-MM-DD-slug.bib   # optional; see step 5

   toc:
     - name: Section One
     - name: Section Two
       subsections:
         - name: Subsection
   ---
   ```

4. **Add figures.** Drop images into `assets/img/posts/<slug>/` and embed with:

   ```liquid
   {% include figure.liquid loading="eager" path="assets/img/posts/<slug>/fig.png" class="img-fluid rounded z-depth-1" %}
   <div class="caption">Caption text here.</div>
   ```

   For side-by-side figures, wrap two `figure.liquid` includes in a Bootstrap row (see the worked example).

5. **Add citations** (distill layout only). Put your BibTeX file at `assets/bibliography/<slug>.bib` (not `_bibliography/` — that path is for the disabled jekyll-scholar publications page). Reference it from frontmatter as `bibliography: <slug>.bib` and cite inline:

   ```html
   ...as shown in our paper <d-cite key="entrykey"></d-cite>.
   ```

   The references list is generated automatically — do not write a manual `## References` section.

---

## Previewing locally

```bash
docker compose up
```

The site serves at **<http://127.0.0.1:8080/blog/>**. Edits to markdown, layouts, and `_data/` hot-reload via livereload. Changes to `_config.yml` require:

```bash
docker compose restart jekyll
```

**Future-dated posts are hidden by default.** If your post's `date:` is past today, it appears immediately. If it's in the future, either set the date to today/past while drafting, or add `future: true` to `_config.yml` to preview future-dated drafts locally (remove before merging if you want production to honor the publish date).

To stop the server:

```bash
docker compose down
```

---

## Deploying

Push to `main`. The workflow at [`.github/workflows/deploy.yml`](.github/workflows/deploy.yml) builds with `JEKYLL_ENV=production` and deploys `_site/` to the `gh-pages` branch via `JamesIves/github-pages-deploy-action`. Live within ~2–3 minutes of the workflow finishing.

A few notes:

- The workflow rewrites `giscus.repo` in `_config.yml` to the current GitHub repository at build time; leave the field empty in source.
- `baseurl: /blog` in `_config.yml` is what makes URLs resolve under `kilian-group.github.io/blog/`. Don't strip it unless you switch to a user/org site (`kilian-group.github.io`).
- Post permalinks are `/:year/:title/` (the `/blog/` prefix comes from `baseurl`, not the permalink). Same applies to tag/category/year archive routes.

---

## Repo layout cheatsheet

| Path | Purpose |
| --- | --- |
| `_posts/` | Blog posts. One markdown file per post. |
| `_pages/about.md` | Landing page (renders the post feed at `/`). |
| `_pages/404.md` | 404 page. |
| `_bibliography/*.bib` | BibTeX files referenced from distill posts. |
| `assets/img/posts/<slug>/` | Per-post image assets. |
| `_config.yml` | Site identity, URL config, feature toggles, plugin list. |
| `.github/workflows/deploy.yml` | CI build + deploy. |
| `README_template.md` | Upstream al-folio theme reference (features, plugin catalog). |
