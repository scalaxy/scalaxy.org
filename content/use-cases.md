---
date: 2026-08-06
lastmod: 2026-08-17
title: "Use Cases"
description: "Where Scalaxy fits: workloads, deployment shapes, and the problems it solves."
kicker: "Use cases"
weight: 20
---

Scalaxy is a graph database written in Common Lisp. A database stores
property graphs queried with openCypher, replicates writes to configured
peers, and serves an HTTP console from every node. The same process also
exposes a key/value API.

## Graph workloads

Graph records use the same ring, durability log, and replication path as
key/value records. There is no separate graph service to operate:

- **Relationship queries**: `MATCH (a:Person)-[:KNOWS]->(b:Person)`,
  `OPTIONAL MATCH`, variable-length paths (`-[:TRIP*1..3]->`), named
  paths, and full aggregation over grouped results.
- **Entity stores with links**: profiles, orders, devices, and other linked
  entities. `CREATE`, `MERGE`, `SET`, `REMOVE`, and `DETACH DELETE` operate
  on the same replicated ring.
- **Bulk analysis**: the shipped NYC taxi benchmark loads 2.9M
  relationships in ~20 s on a laptop, so large scans are tractable on a
  single node and shard across a cluster like any other data.

## Session and user data

Store sessions, profiles, and per-user state behind an API gateway. Keys such
as `user:42` are assigned to ring owners automatically. Configure
`SCALAXY_REPLICATE_TO` to keep a synchronous copy on another node:

```sh
curl -X PUT http://scalaxy:8080/api/keys/user:42 -d '{"value":"{\"theme\":\"dark\"}"}'
curl http://scalaxy:8080/api/keys/user:42
```

## Feature flags and configuration

Keep flags and runtime settings in one store. Prefix scans are available in the console, and values are shown as UTF-8 and hex.

## Cache and hot storage tier

Use it as a shared cache when restart recovery matters. A write is appended
to the durability log before acknowledgement. Evict entries with `delete`.

## Edge and embedded deployments

The SBCL binary can run on small machines, single-board computers, and
air-gapped networks. The HTTP console is served by the database process,
so there is no frontend build to maintain.

## Microservice shared state

For a small fleet of services, the key/value API provides shared state without
a separate cache or database cluster:

{{< callout type="note" title="Embedded mode" >}}
The in-process cluster API (`make-cluster`) is also usable as an embedded library. It runs a database inside your Lisp process.
{{< /callout >}}

## Demos, labs, and education

The repository includes a test suite with 9,018 checks, an openCypher TCK
runner, and a small binary protocol. `make test` exercises TCP and HTTP,
replication, failover, crash recovery, and the Cypher engine end to end.

## When to consider something else

- **Deep analytics over huge scans**: aggregation materializes rows today.
  For terabyte-scale OLAP, use a columnar engine.
- **Multi-datacenter consensus**: replication is synchronous and
  leader-based. For cross-region strong consistency, use a consensus system.
