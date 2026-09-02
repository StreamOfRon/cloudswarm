# Cloudswarm — Hugo personal site/blog, GitHub Actions → Cloudflare Pages, Decap CMS

## Context

Create a new Hugo static site (personal website/blog titled "Cloudswarm") in the empty directory `/home/rmiller/Work/cloudswarm`. Repo will be `github.com/StreamOfRon/cloudswarm`, deployed via GitHub Actions to a Cloudflare Pages project named `cloudswarm` (site URL `https://cloudswarm.pages.dev/`). Decap CMS wired at `/admin/` for content edits, reusing the existing OAuth proxy from the cocoscanines project (`https://gh-oauth-proxy.ron-250.workers.dev`, app_id `Ov23li4N36XuKCkM1a75`). Theme is hand-rolled (no Hugo theme dependency): dark terminal aesthetic, green-on-black, monospace. Reference patterns live in `../cocoscanines` (verified this session); its deploy.yml targets GitHub Pages — we replace the deploy step with `cloudflare/wrangler-action`.

Hugo v0.165.0+extended is installed locally.

## Approach

### Step 1 — Project scaffolding

Create in `/home/rmiller/Work/cloudswarm`:

- `package.json` — copy of cocoscanines': scripts `dev: hugo server -D`, `build: hugo --minify`, name `cloudswarm`, private.
- `.gitignore` — exact copy of `../cocoscanines/.gitignore` (`/public/`, `/resources/`, `.hugo_build.lock`, node_modules, OS/IDE files).
- `config.yaml`:
  ```yaml
  baseURL: "https://cloudswarm.pages.dev/"
  locale: en-us
  title: "Cloudswarm"
  pygmentsUseClasses: false

  markup:
    goldmark:
      renderer:
        unsafe: true
    highlight:
      style: monokai

  taxonomies:
    tag: tags

  params:
    description: "Personal site and blog of StreamOfRon."
    github: "https://github.com/StreamOfRon"
  ```
- `archetypes/default.md` — copy of `../cocoscanines/archetypes/default.md`.
- `archetypes/posts.md` — new (no equivalent exists; post front matter):
  ```
  ---
  title: "{{ replace .Name "-" " " | title }}"
  date: {{ .Date }}
  draft: true
  tags: []
  description: ""
  ---
  ```

### Step 2 — Content

- `content/_index.md` — front matter: `title: "Cloudswarm"`, `description: "Personal site and blog."`, plus `hero_headline: "cloudswarm"` and `hero_subtext: "personal site & blog"` (consumed by home layout, editable via Decap).
- `content/about.md` — front matter `title: "About"`, `description: "About me."`, body: a short placeholder paragraph.
- `content/posts/_index.md` — front matter `title: "Posts"`, `description: "Blog posts."`.
- `content/posts/hello-world.md` — first published post: `title: "Hello, World"`, `date: 2026-09-01`, `draft: false`, `tags: ["meta"]`, one-paragraph body announcing the site.

### Step 3 — Layouts

Pattern copied from cocoscanines (hand-rolled, `assets/css` piped through `resources.Get | minify | fingerprint`). New code only where the reference has no blog equivalent.

- `layouts/_default/baseof.html` — copy of `../cocoscanines/layouts/_default/baseof.html` verbatim.
- `layouts/partials/head.html` — copy of cocoscanines' head.html with these changes:
  - Remove the Google Fonts `<link>` block and the JSON-LD `<script>` block entirely (business-specific; a personal site needs no LocalBusiness schema).
  - Remove the `hero_image` og:image line; keep all other meta/OG/twitter tags unchanged.
  - CSS line unchanged: `{{ $css := resources.Get "css/main.css" | minify | fingerprint }}`.
- `layouts/partials/header.html` — new (cocoscanines' is dog-branding-specific): `<header>` with site title linking to `/` (styled as a terminal prompt, e.g. `ron@cloudswarm:~$` rendered via CSS `::before`, title text from `.Site.Title`) and `<nav>` links: Home `/`, Posts `/posts/`, About `/about/`.
- `layouts/partials/footer.html` — new minimal: `© {{ now.Year }} {{ .Site.Title }}` plus a link to `{{ "index.xml" | relURL }}` labeled "RSS".
- `layouts/index.html` — new home page: hero section using `.Params.hero_headline` / `.Params.hero_subtext` (fallbacks: site title / site description), then a "Recent Posts" section: `{{ range first 5 (where .Site.RegularPages "Section" "posts") }}` listing each post's linked title, `.Date.Format "2006-01-02"`, and `.Description`.
- `layouts/_default/list.html` — new: page title heading, then `{{ range .Pages }}` listing posts as above (title, date, description). Renders both `/posts/` and taxonomy/tag lists.
- `layouts/_default/single.html` — new: `<article>` with `h1` title, date line (`2006-01-02`), tag links (`{{ range .Params.tags }}` → `/tags/{{ . | urlize }}/`), then `{{ .Content }}`.
- `layouts/404.html` — new: minimal "404 — command not found" styled page defining `{{ define "main" }}` (fits baseof).

### Step 4 — Terminal CSS

- `assets/css/main.css` — new, hand-rolled (no existing equivalent). Requirements:
  - Palette: background `#0a0f0a`; body text `#33cc66`; headings and accents `#00ff41`; borders/rules `#1a331a`; dimmed secondary text `#2a8a4a`; code block background `#0d140d`.
  - Font stack: `ui-monospace, "JetBrains Mono", "Cascadia Mono", Menlo, Consolas, monospace` everywhere.
  - Links: `#00cc44`, underline; hover brightens to `#00ff41`. No visited color change.
  - Max content width ~72ch centered; header sticky with bottom border `#1a331a`; nav links separated by `|` or whitespace.
  - Header site title gets a prompt-style `::before` (e.g. content `"$ "` in accent color).
  - `pre` blocks: padding, `#1a331a` border, overflow-x auto; inline `code` dimmer green on `#0d140d`.
  - Post list items: date in dimmed green before title, no bullets.
  - Subtle CRT flavor allowed (e.g. faint text-shadow `0 0 4px rgba(0,255,65,.3)` on headings) but keep it readable; no scanline overlays or animations.

### Step 5 — Decap CMS

- `static/admin/index.html` — exact copy of `../cocoscanines/static/admin/index.html` (loads `decap-cms@3` from unpkg).
- `static/admin/config.yml`:
  ```yaml
  backend:
    name: github
    repo: StreamOfRon/cloudswarm
    branch: main
    base_url: "https://gh-oauth-proxy.ron-250.workers.dev"
    app_id: "Ov23li4N36XuKCkM1a75"

  media_folder: "static/images/uploads"
  public_folder: "/images/uploads"

  collections:
    - name: "pages"
      label: "Pages"
      files:
        - file: "content/_index.md"
          name: "home"
          label: "Home Page"
          fields:
            - {label: "Title", name: "title", widget: "string"}
            - {label: "Hero Headline", name: "hero_headline", widget: "string"}
            - {label: "Hero Subtext", name: "hero_subtext", widget: "text"}
            - {label: "Description", name: "description", widget: "string"}
        - file: "content/about.md"
          name: "about"
          label: "About"
          fields:
            - {label: "Title", name: "title", widget: "string"}
            - {label: "Description", name: "description", widget: "string"}
            - {label: "Body", name: "body", widget: "markdown"}
    - name: "posts"
      label: "Blog Posts"
      folder: "content/posts"
      create: true
      slug: "{{year}}-{{month}}-{{day}}-{{slug}}"
      fields:
        - {label: "Title", name: "title", widget: "string"}
        - {label: "Date", name: "date", widget: "datetime"}
        - {label: "Draft", name: "draft", widget: "boolean", default: true}
        - {label: "Tags", name: "tags", widget: "list", required: false}
        - {label: "Description", name: "description", widget: "string", required: false}
        - {label: "Body", name: "body", widget: "markdown"}
  ```
  (Posts collection is new — cocoscanines has no folder collection; the `pages` files mirror cocoscanines' pattern.)

### Step 6 — GitHub Actions deploy

- `.github/workflows/deploy.yml`:
  ```yaml
  name: Deploy to Cloudflare Pages

  on:
    push:
      branches: [main]
    workflow_dispatch:

  permissions:
    contents: read

  jobs:
    deploy:
      runs-on: ubuntu-latest
      steps:
        - uses: actions/checkout@v4

        - name: Setup Hugo
          uses: peaceiris/actions-hugo@v3
          with:
            hugo-version: 'latest'
            extended: true

        - name: Build
          run: hugo --minify

        - name: Deploy to Cloudflare Pages
          uses: cloudflare/wrangler-action@v3
          with:
            apiToken: ${{ secrets.CLOUDFLARE_API_TOKEN }}
            accountId: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
            command: pages deploy public --project-name=cloudswarm --branch=main
  ```
  `--branch=main` is explicit so the deploy is a production deploy (CI checkout is detached HEAD; wrangler's branch inference can yield preview deploys otherwise). `wrangler pages deploy` auto-creates the `cloudswarm` Pages project on first run.

### Step 7 — Git init (no remote push)

- `git init -b main`, stage everything, single commit "Initial Cloudswarm site: Hugo + Decap CMS + Cloudflare Pages deploy". Do NOT create the GitHub repo or push — user owns that. Remind user at the end:
  1. Create `github.com/StreamOfRon/cloudswarm` and push.
  2. Add repo secrets `CLOUDFLARE_API_TOKEN` (API token with Cloudflare Pages:Edit permission) and `CLOUDFLARE_ACCOUNT_ID`.
  3. First push triggers the workflow, which creates the Pages project.

## Critical files & anchors

- `../cocoscanines/.github/workflows/deploy.yml` — Hugo setup steps to reuse verbatim; replace upload/deploy-pages steps with wrangler-action.
- `../cocoscanines/static/admin/config.yml` — Decap backend block to copy (change `repo:` only); posts collection is new.
- `../cocoscanines/layouts/partials/head.html` — copy minus Google Fonts and JSON-LD blocks.
- `assets/css/main.css` — the only substantially novel file; palette values above are the contract.

## Verification

Local, in `/home/rmiller/Work/cloudswarm`:

1. `hugo --minify` exits 0 and produces `public/` containing `index.html`, `posts/index.html`, `posts/hello-world/index.html`, `about/index.html`, `admin/index.html`, `admin/config.yml`, `index.xml`.
2. `grep -q 'gh-oauth-proxy.ron-250.workers.dev' public/admin/config.yml` — backend wired.
3. `hugo server -D` then browser-check `http://localhost:1313/`: dark green-on-black rendering, header nav (Home/Posts/About), hero text, "Recent Posts" shows "Hello, World" with date `2026-09-01`; click through to the post (title, date, `meta` tag link, body render); `/posts/` and `/about/` render; `/admin/` serves the Decap loader page (script tag for decap-cms@3 visible; actual login requires the pushed repo + OAuth authorization).
4. CSS check: view-source or DevTools shows the fingerprinted stylesheet from `/css/main.*.css` and computed body background `#0a0f0a`.
5. End-to-end deploy (after user creates repo + secrets): push to `main` → Actions workflow green → `https://cloudswarm.pages.dev/` serves the site; `/admin/` login works after authorizing the OAuth app on the new repo.

## Assumptions & contingencies

- Reused OAuth app (`Ov23li4N36XuKCkM1a75`) can be authorized for `StreamOfRon/cloudswarm` by the repo owner on first `/admin/` login. If GitHub denies access (org-level OAuth restrictions — unlikely for a personal repo), fallback: create a new GitHub OAuth App + Worker proxy and update the two `base_url`/`app_id` lines in `static/admin/config.yml`.
- Cloudflare Pages project `cloudswarm` does not yet exist; wrangler creates it. If it already exists with different settings, the deploy still works but the URL may differ if a custom domain is attached.
- `hugo-version: 'latest'` in CI matches cocoscanines' working setup; local hugo is v0.165.0, so no version-sensitive features are used (only `resources.Get|minify|fingerprint`, `where`, `first`, `now.Year` — all long-stable).
