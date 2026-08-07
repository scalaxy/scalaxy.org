---
date: 2026-08-06
lastmod: 2026-08-06
title: "Building & testing"
description: "Compile with ASDF, run the test suite, and understand the project layout."
weight: 20
---

## System definitions

The project is a standard ASDF system:

```sh
sbcl --non-interactive \
  --eval '(require :asdf)' \
  --eval '(asdf:load-asd (truename "scalaxy.asd"))' \
  --eval '(asdf:load-system "scalaxy")'
```

Or use the Makefile:

```sh
make build    # compile the system
make test     # run the full test suite
```

## Test suite

```sh
sbcl --script scripts/run-tests.lisp
```

The suite is intentionally **dependency-free** — plain assertions, no test framework — so it runs anywhere SBCL runs, including CI. It exercises real sockets (TCP and HTTP on ephemeral ports) and real crash-recovery behavior through log replay.

## Project layout

```
scalaxy.asd            ASDF system definition
src/
  package.lisp         package definition
  util.lisp            FNV-1a + SplitMix64 hashing, octet/string helpers
  protocol.lisp        binary wire format + framing
  storage.lisp         durable key/value store (append-only log + replay)
  consistent-hash.lisp virtual-node consistent hashing ring
  replication.lisp     leader op log
  node.lisp            storage node + request dispatch + replication
  tcp.lisp             SBCL TCP server/client + hostname resolution
  json.lisp            dependency-free JSON encoder/decoder
  http.lisp            minimal HTTP/1.1 server/client
  web.lisp             web console: dashboard, REST API, /healthz
  gateway.lisp         cluster gateway: ring routing, failover, status
  cluster.lisp         in-process cluster (routing + replication)
  api.lisp             high-level client API
  main.lisp            node entry point (CLI + SCALAXY_* env config)
web/                   console assets (HTML/CSS/JS)
tests/                 test suite
deploy/                Docker, docker-compose, Kubernetes manifests
```

## Style

- Portable ANSI Common Lisp; SBCL-specific code is guarded with `#+sbcl` and limited to `tcp.lisp`, `http.lisp`, and `main.lisp`.
- Run the suite after any change; keep it green before opening a PR.
- See [Contributing](/docs/contributing/) for the full contribution workflow.
