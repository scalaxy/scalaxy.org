---
date: 2026-08-06
lastmod: 2026-08-17
title: "Use Cases"
description: "Where Scalaxy fits: workloads, deployment shapes, and the problems it solves."
kicker: "Use cases"
weight: 20
---

Scalaxy is a small, durable **graph and key/value** database: keys shard
across nodes with consistent hashing, writes replicate synchronously,
every database also holds a property graph queried with the openCypher
language, and every node serves its own console.  These are the workloads
it fits.

## Session and user data

Store sessions, profiles, and per-user state behind an API gateway. Keys like `user:42` shard across the ring automatically, and with `SCALAXY_REPLICATE_TO` configured, losing any single node costs nothing:

```sh
curl -X PUT http://scalaxy:8080/api/keys/user:42 -d '{"value":"{\"theme\":\"dark\"}"}'
curl http://scalaxy:8080/api/keys/user:42
```

## Feature flags and configuration

Keep flags and runtime settings in one store. Reads are cheap, the console scans by prefix, and values show up as both UTF-8 and hex.

## Cache and hot storage tier

Use it as a shared cache that survives restarts: every write hits the append-only log first, so nothing vanishes when a node reboots. Evict with `delete` when you need to.

## Edge and embedded deployments

One SBCL binary, no dependencies: it runs without trouble on small machines, single-board computers, and air-gapped networks. The console comes with the database, so there is no separate frontend to deploy.

## Microservice shared state

A small shared store for a fleet of services, without running a database cluster:

{{< callout type="note" title="Embedded mode" >}}
The in-process cluster API (`make-cluster`) is also usable as an embedded library — a database inside your own Lisp process.
{{< /callout >}}

## Demos, labs, and education

Nothing to install, a test suite with 9,018 checks, an openCypher TCK
harness, and a protocol you can read in one sitting make it easy to teach
and experiment with. `make test` drives real TCP connections,
replication, failover, crash recovery, and the Cypher engine end to end.

## Graph workloads

Graph data lives in the same store, the same ring, and the same
durability log as keys — no second system to run:

- **Relationship queries** — `MATCH (a:Person)-[:KNOWS]->(b:Person)`,
  `OPTIONAL MATCH`, variable-length paths (`-[:TRIP*1..3]->`), named
  paths, and full aggregation over grouped results.
- **Entity stores with links** — profiles, orders, devices, and
  everything that references everything else: `CREATE`, `MERGE`, `SET`,
  `REMOVE`, and `DETACH DELETE` are all Cypher clauses on the same
  replicated ring.
- **Bulk analysis** — the shipped NYC taxi benchmark loads 2.9M
  relationships in ~20 s on a laptop, so large scans are tractable on a
  single node and shard across a cluster like any other data.

## When to consider something else

- **Deep analytics over huge scans** — the engine is row-materializing for
  aggregation today; for terabyte-scale OLAP, a columnar engine remains a
  better fit.
- **Multi-datacenter consensus** — replication is synchronous and leader-based; for cross-region strong consistency look at consensus systems.
