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

The site is hosted on **Cloudflare Pages** (connected to the GitHub
repository). Configure it with:

- **Build command:** `hugo --minify`
- **Output directory:** `public`

Every push to `main` triggers a Cloudflare Pages build and deploy.

For manual deploys or other static hosts:

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
