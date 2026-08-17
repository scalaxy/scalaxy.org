---
date: 2026-08-06
lastmod: 2026-08-17
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

The suite is intentionally **dependency-free** — plain assertions, no test framework — so it runs anywhere SBCL runs, including CI. It exercises real sockets (TCP and HTTP on ephemeral ports), real crash-recovery behavior through log replay, and the openCypher engine end to end (TCP, gateway, and web layers included).

The **9,018 checks** span 33 groups, from consistent hashing and durable
storage through the graph layer (storage, blobs, persistence, multi-db,
gateway) and the Cypher engine (lexer, parser, AST round-trips, executor,
semantics, updates, plus differential tests against a reference oracle).

A second runner, `scripts/run-tck.lisp`, executes the **openCypher TCK**
(3,897 scenarios from the official conformance suite) and classifies each
scenario as pass, fail, or unsupported; see [Cypher](/docs/cypher/).

## Project layout

```
scalaxy.asd            ASDF system definition
src/
  package.lisp         package definition
  util.lisp            FNV-1a + SplitMix64 hashing, octet/string helpers
  protocol.lisp        binary wire format + framing (+ CYPHER opcode)
  storage.lisp         durable key/value store (append-only log + replay)
  consistent-hash.lisp virtual-node consistent hashing ring
  replication.lisp     leader op log
  node.lisp            storage node + request dispatch + replication
  tcp.lisp             SBCL TCP server/client + hostname resolution
  json.lisp            dependency-free JSON encoder/decoder
  http.lisp            minimal HTTP/1.1 server/client
  web.lisp             web console: dashboard, REST API, /healthz, /api/cypher
  gateway.lisp         cluster gateway: ring routing, failover, status
  cluster.lisp         in-process cluster (routing + replication)
  api.lisp             high-level client API (put/get/scan/cypher)
  main.lisp            node entry point (CLI + SCALAXY_* env config)
  graph.lisp           property-graph storage over the KV store
  db.lisp              multi-database namespacing + entity ids
  codec.lisp           binary codec for graph entities
  cypher/              openCypher engine: lexer, parser, AST, functions,
                       semantics, updates, executor, wire, oracle
web/                   console assets (HTML/CSS/JS)
tests/                 test suite (+ TCK runner)
benchmarks/            Movie Graph and NYC taxi benchmark datasets
deploy/                Docker, docker-compose, Kubernetes manifests
```

## Style

- Portable ANSI Common Lisp; SBCL-specific code is guarded with `#+sbcl` and limited to `tcp.lisp`, `http.lisp`, and `main.lisp`.
- Run the suite after any change; keep it green before opening a PR.
- See [Contributing](/docs/contributing/) for the full contribution workflow.
