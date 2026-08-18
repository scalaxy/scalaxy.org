---
date: 2026-08-06
lastmod: 2026-08-17
title: "Architecture"
description: "Consistent hashing, synchronous replication, durable logs, the graph layer, and the openCypher engine."
weight: 30
---

## Overview

```
+--------------------------------------------------------------+
|            client (REST / Lisp API / Cypher)                  |
+--------------------------------------------------------------+
        | put/get/delete/scan · cypher (binary frames over TCP)
        v
+--------------------------------------------------------------+
|  consistent-hash ring  ->  primary node owns key/entity      |
|  node-put: apply locally, persist to log, replicate to       |
|            next N ring members, await acks                   |
+--------------------------------------------------------------+
|  per-node storage: in-memory hash table + append-only log    |
|  graph layer: nodes, relationships, labels, indexes, blobs   |
+--------------------------------------------------------------+
|  Cypher engine: lexer -> parser -> AST -> semantics -> exec  |
+--------------------------------------------------------------+
|  web console: HTTP server + JSON REST API + /healthz         |
+--------------------------------------------------------------+
```

## Consistent hashing

Keys are hashed with **FNV-1a 64-bit plus a SplitMix64 finalizer** for a uniform spread, then placed on a ring of **virtual nodes** (128 per physical node by default). A key is owned by the first vnode clockwise from its hash.

- Adding or removing a node moves only ~1/N of the keys.
- Because vnode positions are deterministic, every member computes the same ownership — no coordination protocol is required for routing.

## Replication

Writes go through `node-put`, which:

1. applies the mutation to the local store,
2. persists it to the append-only log,
3. forwards a `replicate` message to every configured follower (the next members in ring order),
4. acknowledges the client.

Replicas apply the mutation to their own store **without** re-logging it (the log is a node-local record). Reads that miss the ring owner due to failure fall back to the remaining members, which hold the replicas.

## Storage & durability

Each node keeps an in-memory hash table plus an **append-only log** using the same length-prefixed record format as the network protocol. On startup the log is replayed to reconstruct state; the log filename is a stable `scalaxy.log` in the data directory, so restarts (and container restarts) find the data.

## Graph storage & Cypher engine

On top of the replicated key/value store sits a **property-graph layer**
({{< src "src/graph.lisp" >}}, {{< src "src/db.lisp" >}}, {{< src "src/codec.lisp" >}}):

- **Nodes & relationships** — every node has an id, a label set, and a
  property map; every relationship has an id, a type, start/end node ids,
  and a property map.  Large binary properties spill to blob records.
- **Indexes** — label, type, and adjacency indexes (outgoing/incoming
  relationships per node) keep pattern matching local.
- **Multi-database** — a database is a namespaced partition of the same
  ring; graph entities, keys, and the durability log all live in the one
  replicated store.
- **openCypher engine** ({{< src "src/cypher/" >}}) — a full compiler pipeline in pure
  Common Lisp: lexer → parser → AST (with canonical printer) → semantic
  analysis → executor, plus a wire format and a metacircular reference
  oracle used for differential testing.  The executor is symbolically
  aggregated, so `count(a) * 10 + count(b) * 5` and nested `collect`
  expressions evaluate correctly.

Cypher queries reach the engine over the same binary frames (`CYPHER`
opcode), through the REST API (`POST /api/cypher`), the Lisp client, or
the web-console command line.  Queries route through the cluster gateway
like any other operation.

## Wire protocol

See [Protocol](/docs/protocol/) for the full specification. The same framing carries client requests, replication messages, and log records.

## Web console

Every node runs an HTTP server (default port 8080) that serves the dashboard and the REST API. With `SCALAXY_PEERS` configured, the node acts as a **gateway**: it routes key operations to ring owners over TCP, aggregates status from every peer's `/api/node-status`, and exposes the whole cluster in one console. See [REST API](/docs/rest-api/).

## Consistency model

The graph layer shares the ring's consistency story: a Cypher mutation
that creates or updates entities is applied through `node-put`/`node-delete`
on the ring owner and replicated synchronously, so acknowledged graph
writes survive crashes and fail over to replicas.  `CREATE`, `MERGE`,
`SET`, `REMOVE`, and `DELETE` all flow through the durable log.

## Failure model

| Event | Behavior |
|---|---|
| Node crash | Durability log replayed on restart; acknowledged writes survive. |
| Node unreachable | Reads fail over to replicas; status shows `degraded` / `unreachable`. |
| Node removal (cluster API) | Keys are re-homed to their new ring owners before removal completes. |
| Node rejoin | Ring ownership resumes; cluster returns to `healthy`. |
