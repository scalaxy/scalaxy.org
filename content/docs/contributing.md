---
date: 2026-08-06
lastmod: 2026-08-06
title: "Contributing"
description: "How to contribute code, documentation, and bug reports."
weight: 100
---

## Welcome

Scalaxy is an independent, open-source project. Contributions of code, documentation, tests, and ideas are all welcome.

## Development workflow

1. **Fork & clone** the repository.
2. **Run the tests** before changing anything: `make test`.
3. **Make a focused change** — add tests alongside any behavior change.
4. **Keep the suite green** — `make test` must pass locally.
5. **Open a pull request** describing the change and the tests.

## What to work on

- Bugs found while running the 8,650-check suite.
- The roadmap in the README: snapshot-based catch-up for lagging replicas, a membership/join protocol, multi-key transactions.
- Documentation, the web console, and deployment examples.

## Guidelines

- Pure ANSI Common Lisp where possible; keep SBCL-specific code behind `#+sbcl` and out of the core modules.
- The JSON/HTTP/TCP layers are intentionally dependency-free — please keep them that way.
- Every public function should have a docstring.
- Update the relevant docs page in this portal when you change behavior.

## Code of conduct

Be respectful and constructive. This is a small project — every contributor matters.
