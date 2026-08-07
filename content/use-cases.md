---
date: 2026-08-06
lastmod: 2026-08-06
title: "Use Cases"
description: "Where Scalaxy fits: workloads, deployment shapes, and the problems it solves."
kicker: "Use cases"
weight: 20
---

Scalaxy is a **multi-purpose, cloud-ready distributed key/value database**. Its combination of consistent-hash sharding, synchronous replication, durable append-only storage, and an embedded operations console makes it a practical choice for a wide range of workloads — especially where a single small binary and minimal operational surface matter.

## Session & user data

Store sessions, profiles, and per-user state behind an API gateway. Keys like `user:42` shard across the ring automatically, and with `SCALAXY_REPLICATE_TO` configured, losing any single node costs nothing:

```sh
curl -X PUT http://scalaxy:8080/api/keys/user:42 -d '{"value":"{\"theme\":\"dark\"}"}'
curl http://scalaxy:8080/api/keys/user:42
```

## Feature flags & configuration

A single source of truth for runtime configuration. Read-heavy workloads benefit from the web console's prefix scan and from the ability to inspect values as UTF-8 and hex side by side.

## Cache & hot storage tier

Use Scalaxy as a shared cache layer with stronger durability than an in-memory cache. The append-only log means a restart does not empty the cache; eviction policies can be layered on top with `delete`.

## Edge & embedded deployments

Because Scalaxy is a single SBCL binary with zero external dependencies, it runs comfortably on small machines, single-board computers, and air-gapped networks. The web console is served by the database itself — no separate frontend stack required.

## Microservice shared state

Give a fleet of services a small, well-understood shared store without standing up a heavyweight database cluster:

{{< callout type="note" title="Embedded mode" >}}
The in-process cluster API (`make-cluster`) is also usable as an embedded library — a database inside your own Lisp process.
{{< /callout >}}

## Demos, labs, and education

A zero-dependency install, a 8,650-check test suite, and a clean protocol make Scalaxy easy to teach and experiment with. `make test` exercises real TCP networking, replication, failover, and persistence.

## When to consider something else

- **Complex queries & joins** — Scalaxy is a key/value store; use a relational or document database for relational workloads.
- **Multi-datacenter consensus** — replication is synchronous and leader-based; for cross-region strong consistency look at consensus systems.
- **Massive analytics** — columnar/analytical engines are a better fit for large scans.
