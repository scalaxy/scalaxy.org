# scalaxy.org — Scalaxy website

The [Scalaxy](https://github.com/scalaxy/scalaxy) project website: a **Hugo** static site with the general description, use cases, topologies, and the developer & maintainer documentation portal.

## Develop

```sh
hugo server -D      # http://localhost:1313
```

## Build

```sh
hugo                # static output in public/
```

## Project layout

```
content/            Markdown pages (home, use cases, topologies, docs)
themes/scalaxy/     The site theme (layouts, CSS, JS)
data/features.yml   Feature cards for the home page
public/             Build output (generated, not committed)
deploy/             Deployment helpers
```

## Theme

`themes/scalaxy` is a minimal, enterprise-style light theme:
clean typography, restrained blue accent, dark terminal accent, light
syntax highlighting (chroma `github` style), and a three-column docs
layout with sidebar navigation, "on this page" TOC, prev/next paging,
and prev/next paging.

Layout lookup follows Hugo conventions: `index.html` for the home page,
`docs/single.html` for doc pages, `docs/landing.html` (selected via
`layout: "landing"` in `content/docs/_index.md`) for the docs portal,
`docs/list.html` for section listings such as `/docs/maintainers/`,
and `_default/single.html` for standalone pages (use cases, topologies).

## Docs portal

The documentation lives in `content/docs/` (developer guides) and
`content/docs/maintainers/` (maintainer guide). The docs layout renders a
sidebar, breadcrumbs, and prev/next paging automatically from page `weight`.

## Deploy

The site is hosted on **Cloudflare Pages** (a Pages project — **not** Workers).
It follows the official
[Deploy a Hugo site](https://developers.cloudflare.com/pages/framework-guides/deploy-a-hugo-site/)
framework guide. In the **Set up builds and deployments** section:

| Configuration option | Value                       |
|----------------------|-----------------------------|
| Production branch    | `main`                      |
| Build command        | `hugo --minify`             |
| Build directory      | `public`                    |

Every push to `main` triggers a Cloudflare Pages build and deploy.

> **Workers vs Pages:** if the build log ever shows
> `Executing user deploy command: npx wrangler deploy` followed by
> `Missing entry-point to Worker script or to assets directory`, the
> Cloudflare project was created as a **Workers** project (Workers
> Builds). Delete it and create a **Pages** project (Workers & Pages →
> Create application → Pages → Import an existing Git repository) with
> the settings above — Pages deploys the `public` directory natively and
> does not run `wrangler deploy`.

### Hugo version

The guide recommends pinning the Hugo version with a `HUGO_VERSION`
environment variable. Set it in the Pages project under
**Settings → Environment variables** for **both** the Production and the
Preview environment (the guide explicitly requires both for preview
deployments):

```
HUGO_VERSION = 0.157.0
```

### Base URL note

The guide shows `hugo -b $CF_PAGES_URL` so absolute URLs follow the Pages
deployment URL. This site uses **relative URLs** (`relURL`) for all links
and assets and has a custom domain (`scalaxy.org`), so the build command
keeps `baseURL = "https://scalaxy.org/"` from `hugo.toml`. Adding
`-b $CF_PAGES_URL` would rewrite canonical/`og:url`/sitemap URLs to the
`*.pages.dev` domain in production, which is why it is intentionally not
used here.

### Wrangler CLI

The repository is configured for [Wrangler](https://developers.cloudflare.com/workers/wrangler/)
(`wrangler.toml`, project `scalaxy-org`). After `npm install`:

```sh
npm run build       # hugo --minify
npm run preview     # serve ./public locally via wrangler pages dev
npm run deploy      # build + wrangler pages deploy public --project-name scalaxy-org
```

Requires `wrangler login` (or `CLOUDFLARE_API_TOKEN` / `CLOUDFLARE_ACCOUNT_ID`
env vars) for `deploy`. An optional GitHub Actions workflow
(`.github/workflows/deploy.yml`) is included for push-based deploys — only
use it if you are **not** using the Cloudflare Pages Git integration.

### Other static hosts

```sh
hugo --minify && rsync -av public/ user@host:/var/www/scalaxy.org/
```

## Notes

- Raw HTML (SVG topology diagrams) is enabled via
  `[markup.goldmark.renderer] unsafe = true` in `hugo.toml`.
- The site carries the non-affiliation notice committed to in the GitHub
  Trust & Safety response: Scalaxy is an independent open source distributed
  database project associated with scalaxy.org (registered 2014) and is not
  affiliated with, sponsored by, endorsed by, authorized by, or operated by
  Scalaxy B.V. or scalaxy.com.
