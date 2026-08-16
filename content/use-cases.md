---
date: 2026-08-06
lastmod: 2026-08-06
title: "Use Cases"
description: "Where Scalaxy fits: workloads, deployment shapes, and the problems it solves."
kicker: "Use cases"
weight: 20
---

Scalaxy is a small, durable key/value database that shards keys, replicates writes, and serves its own console. If that is what you need, without standing up a database cluster, these are the workloads it fits.

## Session & user data

Store sessions, profiles, and per-user state behind an API gateway. Keys like `user:42` shard across the ring automatically, and with `SCALAXY_REPLICATE_TO` configured, losing any single node costs nothing:

```sh
curl -X PUT http://scalaxy:8080/api/keys/user:42 -d '{"value":"{\"theme\":\"dark\"}"}'
curl http://scalaxy:8080/api/keys/user:42
```

## Feature flags & configuration

Keep flags and runtime settings in one store. Reads are cheap, the console scans by prefix, and values show up as both UTF-8 and hex.

## Cache & hot storage tier

Use it as a shared cache that survives restarts: every write hits the append-only log first, so nothing vanishes when a node reboots. Evict with `delete` when you need to.

## Edge & embedded deployments

One SBCL binary, no dependencies: it runs without trouble on small machines, single-board computers, and air-gapped networks. The console comes with the database, so there is no separate frontend to deploy.

## Microservice shared state

A small shared store for a fleet of services, without running a database cluster:

{{< callout type="note" title="Embedded mode" >}}
The in-process cluster API (`make-cluster`) is also usable as an embedded library — a database inside your own Lisp process.
{{< /callout >}}

## Demos, labs, and education

Nothing to install, a test suite with 8,654 checks, and a protocol you can read in one sitting make it easy to teach and experiment with. `make test` drives real TCP connections, replication, failover, and crash recovery.

## When to consider something else

- **Complex queries & joins** — Scalaxy is a key/value store; use a relational or document database for relational workloads.
- **Multi-datacenter consensus** — replication is synchronous and leader-based; for cross-region strong consistency look at consensus systems.
- **Massive analytics** — columnar/analytical engines are a better fit for large scans.
