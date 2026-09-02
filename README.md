# cloudswarm

Personal site & blog. Hugo → GitHub Actions → Cloudflare Pages
(<https://cloudswarm.pages.dev/>), with [Decap CMS](https://cloudswarm.pages.dev/admin/)
for browser-based editing.

## Writing a new post

```sh
hugo new content posts/my-topic.md     # scaffolds archetypes/posts.md
npm run dev                            # preview at http://localhost:1313 (-D shows drafts)
```

Then edit the file and publish (see below). Front matter that matters:

```yaml
---
title: "My Topic"            # page title, <title>, OG tags
date: 2026-09-15T08:00:00-07:00  # sort key + publish time (see Scheduling)
draft: true                  # remove or set false to publish
tags: ["redis"]              # optional → /tags/<tag>/ pages
description: "One-liner."    # shown in post lists + RSS/OG; fill it in
---
```

The filename becomes the URL: `content/posts/my-topic.md` → `/posts/my-topic/`.
If you rename later, add `aliases: ["/posts/my-topic/"]` so old links redirect.

## Creating a new page

Any markdown file at `content/` root is a page — `content/now.md` renders at
`/now/`. Minimal front matter:

```yaml
---
title: "Now"
description: "What I'm working on lately."
---
```

To make it reachable from the nav, add a link in
`layouts/partials/header.html`. (Pages outside `content/posts/` won't appear in
Decap's "Blog Posts" collection; add them to the "Pages" list in
`static/admin/config.yml` if you want CMS editing.)

## Publishing

Push to `main` — the Actions workflow builds and deploys in ~40 s:

```sh
git add content/posts/my-topic.md
git commit -m "post: my topic"
git push origin main
```

Or use `/admin/` (GitHub login). Decap commits directly to `main`, so every
save there is also a production deploy.

## Publishing later than now (this site is intermittent)

- **Drafts are safe.** `draft: true` pages are excluded from production builds
  entirely — no URL, no RSS entry. Commit them and walk away.
- **Future-dated posts are too.** Hugo doesn't build pages whose `date` is in
  the future. `date: 2026-09-15 08:00` + `draft: false` + push = the post
  appears on its own once that time passes.
- **The trigger is a rebuild.** The workflow runs daily at **8:45 AM Pacific**
  (see `timezone` in `.github/workflows/deploy.yml`), which sweeps in anything
  whose date has passed. A post dated *after* 8:45 waits for the **next day's**
  run — date it ≤ 08:45 to land same-morning, or just push anything to force an
  immediate rebuild.
- **Gotcha:** the daily rebuild publishes *dated* posts, not *drafts*. A
  `draft: true` post never goes live until you flip the flag and push.

## Images

Drop files in `static/images/uploads/` (the CMS uploads there automatically)
and reference root-relative: `/images/uploads/photo.png`.

## Contact links

One source of truth: `params.social` in `config.yaml`.

```yaml
params:
  social:
    - id: mastodon                                  # → assets/icons/mastodon.svg
      name: "Mastodon"
      url: "https://macaw.social/@streamofron"
      handle: "streamofron@macaw.social"
```

- Every page footer renders the glyph row (`layouts/partials/social.html`,
  mode `icons`); `{{< social >}}` in any content body renders glyphs **and**
  handles (used by `/about/`). Keep that shortcode line if you edit About in
  the CMS — it is what renders the handles there.
- Glyphs are Simple Icons (CC0) paths vendored under `assets/icons/<id>.svg`,
  inlined at build time with `fill="currentColor"`, so they inherit the theme
  green instead of brand colours. New network = drop in the SVG + add the entry.
- A `social` entry whose `id` has no matching icon **fails the build** — no
  silently missing glyphs.
- Links carry `rel="me"`, which is how Mastodon verifies profile links.

## Comments & analytics

Both are config-gated: absent config = nothing loads, no third-party calls.

**Giscus** (comments via GitHub Discussions — free, readers sign in with
GitHub). One-time setup:

1. Repo **Settings → General → Discussions** → enable.
2. Install the app: <https://github.com/apps/giscus> → *Only select
   repositories* → `cloudswarm`.
3. On <https://giscus.app>, enter `StreamOfRon/cloudswarm`; it prints four
   values. Paste into `config.yaml`:

   ```yaml
   params:
     giscus:
       repo: "StreamOfRon/cloudswarm"
       repoId: "R_kgDO..."          # from giscus.app
       category: "Announcements"    # a Discussions category
       categoryId: "DIC_kgDO..."    # from giscus.app
   ```

`layouts/partials/comments.html` then renders at the bottom of every post
(`data-mapping: pathname` — one discussion per post URL).

**Cloudflare Web Analytics** (free, cookieless, no banner needed): Cloudflare
dashboard → *Web Analytics* → *Add site* → pick `cloudswarm.pages.dev` → copy
the token → `config.yaml`:

```yaml
params:
  cfAnalyticsToken: "0a1b2c3d..."
```

`layouts/partials/analytics.html` injects the beacon in `<head>` on every page.

## Reference

| Path | Role |
|---|---|
| `content/posts/` | blog posts (Decap "Blog Posts" collection) |
| `content/*.md` | static pages; `_index.md` = home hero copy |
| `archetypes/` | front-matter scaffolds for `hugo new` |
| `layouts/` | all templates — terminal theme markup |
| `assets/css/main.css` | the whole stylesheet (hand-rolled, no theme) |
| `assets/icons/` | brand glyphs for `params.social` (Simple Icons, CC0) |
| `layouts/partials/social.html` | renders contact links (footer + `{{< social >}}`) |
| `layouts/partials/comments.html` | Giscus embed (only if `params.giscus` set) |
| `layouts/partials/analytics.html` | CF Analytics beacon (only if `params.cfAnalyticsToken` set) |
| `static/admin/` | Decap CMS loader + config |
| `.github/workflows/deploy.yml` | build + deploy to Cloudflare Pages |
