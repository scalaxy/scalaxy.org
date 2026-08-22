---
date: 2026-08-06
lastmod: 2026-08-21
title: "Architecture"
description: "Consistent hashing, synchronous replication, S3-backed lazy storage, aggregate summaries, durable outbox, and the openCypher engine."
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
|  per-node storage backend:                                   |
|    local: in-memory hash table + append-only log             |
|    S3: packed immutable segments + local persistent cache    |
|        + sidecar metadata (Bloom filters) + encryption       |
|  graph layer: nodes, relationships, labels, indexes, blobs   |
+--------------------------------------------------------------+
|  Cypher engine: lexer -> parser -> AST -> semantics -> exec  |
|  aggregate pushdown: type-counts · sums · endpoint pairs     |
+--------------------------------------------------------------+
|  web console: HTTP server + JSON REST API + /healthz         |
+--------------------------------------------------------------+
```

## Storage backends

Scalaxy supports two storage backends selected by configuration:

### Local append-only log

Each node keeps an in-memory hash table plus an **append-only log** using
the same length-prefixed record format as the network protocol. On startup
the log is replayed to reconstruct state.

### S3-compatible object storage

For datasets larger than memory, Scalaxy stores data in an **S3-compatible
object store** (Garage, MinIO, AWS S3) with a rich set of optimisations:

- **Packed immutable segments**: up to 20 000 records per object,
  codec-encoded as a single blob. Segments are write-once; deletes are
  recorded as tombstone markers.
- **Lazy loading**: nodes boot without materialising the full key space.
  An in-memory index maps each key to its segment and byte range; values
  are read on demand from the local persistent cache.
- **Local persistent cache**: every fetched S3 object is cached on local
  disk under its SHA-256 digest. After warm-up the cache hit rate reaches
  100 %, so steady-state reads never touch the network.
- **Sidecar metadata**: per-segment `.idx` files store key/offset/type
  triples, record counts, key ranges, and 256-bit Bloom filters.
  Sidecars are validated on load and rebuilt if corrupt.
- **Aggregate summaries**: type counts, per-property sums, label-ID sets,
  top-k lists, and endpoint-pair tables accelerate count, sum, avg, and
  labeled relationship queries without decoding any packed records.
- **Authenticated encryption at rest**: when `SCALAXY_S3_ENCRYPTION_KEY`
  is set, every object body is encrypted with HMAC-SHA256 CTR-mode
  encryption before upload (format: SCX1 ‖ nonce ‖ ciphertext ‖ tag).

See [S3 object storage](/docs/s3-storage/) for the full specification.

## Consistent hashing

Keys are hashed with **FNV-1a 64-bit plus a SplitMix64 finalizer** for a uniform spread, then placed on a ring of **virtual nodes** (128 per physical node by default). A key is owned by the first vnode clockwise from its hash.

- Adding or removing a node moves only ~1/N of the keys.
- Because vnode positions are deterministic, every member computes the same ownership: no coordination protocol is required for routing.

## Replication

Writes go through `node-put`, which:

1. applies the mutation to the local store,
2. persists it to the durability log or S3 batch queue,
3. forwards a `replicate` message to every configured follower (the next members in ring order),
4. acknowledges the client.

Failed replications are queued in a **durable outbox** that persists across leader restarts. A background thread retries undelivered messages every 5 seconds, ensuring followers catch up after returning to service.

Replicas apply the mutation to their own store **without** re-logging it (the log is a node-local record). Reads that miss the ring owner due to failure fall back to the remaining members, which hold the replicas.

## Graph storage & Cypher engine

On top of the replicated key/value store sits a **property-graph layer**:

- **Nodes & relationships**: every node has an id, a label set, and a
  property map; every relationship has an id, a type, start/end node ids,
  and a property map. Large binary properties spill to blob records.
- **Indexes**: label, type, and adjacency indexes (outgoing/incoming
  relationships per node) keep pattern matching local.
- **Multi-database**: a database is a namespaced partition of the same
  ring; graph entities, keys, and the durability log all live in the one
  replicated store.
- **openCypher engine**: a full compiler pipeline in pure
  Common Lisp: lexer → parser → AST (with canonical printer) → semantic
  analysis → executor, plus a wire format and a metacircular reference
  oracle used for differential testing. The executor is symbolically
  aggregated, so `count(a) * 10 + count(b) * 5` and nested `collect`
  expressions evaluate correctly.

Cypher queries reach the engine over the same binary frames (`CYPHER`
opcode), through the REST API (`POST /api/cypher`), the Lisp client, or
the web-console command line. Queries route through the cluster gateway
like any other operation.

## Aggregate pushdown

Scalar relationship aggregates (COUNT, SUM, AVG over relationship
properties) are pushed down from the gateway to individual nodes via the
`op-aggregate` opcode. Each node computes its owned-held contribution from
in-memory summary tables — type-counts, per-property sums, and
endpoint-pair tables — without decoding any packed segment records.

The gateway sums peer responses and divides by the replication breadth to
produce cluster-wide unique aggregates. This makes labeled count and sum
queries return in **single-digit milliseconds** even on multi-million-edge
graphs.

## Wire protocol

See [Protocol](/docs/protocol/) for the full specification. The same framing carries client requests, replication messages, and log records.

## Web console

Every node runs an HTTP server (default port 8080) that serves the dashboard and the REST API. With `SCALAXY_PEERS` configured, the node acts as a **gateway**: it routes key operations to ring owners over TCP, aggregates status from every peer's `/api/node-status`, and exposes the whole cluster in one console. See [REST API](/docs/rest-api/).

## Consistency model

The graph layer shares the ring's consistency story: a Cypher mutation
that creates or updates entities is applied through `node-put`/`node-delete`
on the ring owner and replicated synchronously, so acknowledged graph
writes survive crashes and fail over to replicas.  `CREATE`, `MERGE`,
`SET`, `REMOVE`, and `DELETE` all flow through the durable log or S3
write path.

Deletes are immediately visible in lazy mode: the key is removed from the
in-memory index before the tombstone is persisted, so subsequent queries
observe the deletion without waiting for a restart.

## Failure model

| Event | Behavior |
|---|---|
| Node crash | Durability log replayed on restart; acknowledged writes survive. S3 objects are authoritative. |
| Node unreachable | Reads fail over to replicas; status shows `degraded` / `unreachable`. |
| Node removal (cluster API) | Keys are re-homed to their new ring owners before removal completes. |
| Node rejoin | Ring ownership resumes; cluster returns to `healthy`. Durable outbox delivers missed replications. |
| Corrupt cache entry | Detected by magic-header validation; silently deleted and re-fetched from S3. |
