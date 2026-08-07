---
date: 2026-08-06
lastmod: 2026-08-06
title: "Architecture"
description: "Consistent hashing, synchronous replication, durable logs, and the web console."
weight: 30
---

## Overview

```
+--------------------------------------------------------------+
|                    client (REST / Lisp API)                   |
+--------------------------------------------------------------+
        | put/get/delete/scan (binary frames over TCP)
        v
+--------------------------------------------------------------+
|  consistent-hash ring  ->  primary node owns key             |
|  node-put: apply locally, persist to log, replicate to       |
|            next N ring members, await acks                   |
+--------------------------------------------------------------+
|  per-node storage: in-memory hash table + append-only log    |
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

## Wire protocol

See [Protocol](/docs/protocol/) for the full specification. The same framing carries client requests, replication messages, and log records.

## Web console

Every node runs an HTTP server (default port 8080) that serves the dashboard and the REST API. With `SCALAXY_PEERS` configured, the node acts as a **gateway**: it routes key operations to ring owners over TCP, aggregates status from every peer's `/api/node-status`, and exposes the whole cluster in one console. See [REST API](/docs/rest-api/).

## Failure model

| Event | Behavior |
|---|---|
| Node crash | Durability log replayed on restart; acknowledged writes survive. |
| Node unreachable | Reads fail over to replicas; status shows `degraded` / `unreachable`. |
| Node removal (cluster API) | Keys are re-homed to their new ring owners before removal completes. |
| Node rejoin | Ring ownership resumes; cluster returns to `healthy`. |
