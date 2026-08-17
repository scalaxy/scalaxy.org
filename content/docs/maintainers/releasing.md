---
date: 2026-08-06
lastmod: 2026-08-17
title: "Releasing"
description: "The release process: versioning, tags, container images, and checks."
weight: 120
---

## Versioning

Scalaxy follows **semantic versioning** (`MAJOR.MINOR.PATCH`). The version lives in `src/web.lisp` (`+version+`), the ASDF system definition (`scalaxy.asd`), the web-console asset cache keys (`web/index.html`), the README badge, and `CITATION.cff`; the website mirrors it in `hugo.toml` (`params.version`) and `package.json`.

## Release checklist

1. **Update the version** in `scalaxy.asd` and `src/web.lisp`.
2. **Run the full suite** locally and in CI:

   ```sh
   make test
   ```

3. **Build the container image** and smoke-test it:

   ```sh
   docker build -t scalaxy:$VERSION .
   docker run -p 8080:8080 scalaxy:$VERSION
   curl localhost:8080/healthz
   ```

4. **Verify a 3-node cluster** (docker-compose or Kubernetes) — write keys, stop one node, confirm zero data loss.
5. **Tag** the release:

   ```sh
   git tag -a v$VERSION -m "Scalaxy v$VERSION"
   git push origin v$VERSION
   ```

6. **Publish** the container image and write release notes summarizing changes and upgrade notes.

## CI

The repository ships a GitHub Actions workflow (`.github/workflows/ci.yml`) that installs SBCL and runs `scripts/run-tests.lisp` on every push. The Docker build also runs the full suite at image-build time, so a release build that fails tests cannot produce an image.

## Rolling back

Because the wire protocol is versioned by the code, roll back by reverting to the previous tag and rebuilding. The durability log format is stable within a major version; check release notes for any migration steps.
