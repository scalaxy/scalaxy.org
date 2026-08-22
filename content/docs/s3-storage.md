---
date: 2026-08-21
lastmod: 2026-08-21
title: "S3 object storage"
description: "Packed immutable segments, lazy loading, ownership-filtered aggregates, authenticated encryption, and self-healing caches on S3-compatible object stores."
weight: 32
---

## Overview

Scalaxy's **S3 storage backend** replaces the local append-only log with an
S3-compatible object store (Garage, MinIO, AWS S3).  Every node maps to its
own bucket and prefix namespace.  Data is organised as:

| Object kind | Key format | Contents |
|---|---|---|
| Packed segments | hex(`@batch:<ts>-<seq>`) | Immutable batches of up to 20 000 records |
| Ordinary objects | hex(key) | Individual key/value pairs written outside batch mode |
| Tombstones | hex(`@tombstone:<key>:<ts>`) | Deletion markers that survive segment replay |
| Sidecar indexes | sha256(relative)`.idx` | Per-segment key/offset/type metadata with Bloom filters |

## Lazy loading

Nodes boot without materialising the full key space.  Instead,
`%s3-load-lazy` builds an **in-memory index** (`key → segment, start, end`)
from sidecar files and LIST results.  Individual values are read on demand
via `%s3-get-cached-range`, which reads only the needed byte range from the
local persistent cache.

```
startup:  LIST (marker-paginated) → classify objects → build lazy-index
          replay segments via sidecars (skip re-decode of unchanged data)
read:     lazy-index lookup → get-cached-range(segment, start, end)
          → served from local persistent cache (zero-copy after warm-up)
```

This means a 5.7 M-entry index is ready in ~4 minutes (dominated by S3
GET throughput), and individual lookups are served from the OS page cache.

## Local persistent cache

Every node maintains a **local disk cache** of all fetched S3 objects.
Cache files are byte-for-byte copies of S3 object bodies stored under
their SHA-256 digests.  After warm-up the cache hit rate reaches **100 %**,
so steady-state reads never touch the network.

Cache eviction uses a budget (`SCALAXY_S3_CACHE_MAX_BYTES`) with oldest-first
eviction when the budget is exceeded.  Cache integrity is verified by
checking a magic header on each cached segment before trusting it;
corrupt entries are silently deleted and re-fetched.

## Sidecar metadata

Each segment has a companion `.idx` file storing per-record metadata:

| Field | Description |
|---|---|
| key | The logical graph key (e.g. `d:taxi:n:g000000001`) |
| offset / end | Byte range within the segment for range reads |
| op | `PUT` or `DELETE` |
| count | Total record count |
| key-range | First and last key (lexicographic) |
| Bloom filter | 256-bit filter over all keys in the segment |

Sidecars are validated on load: the count must match, the key-range must
cover the observed keys, and the Bloom filter must contain every key.
Invalid sidecars are rebuilt from segment replay.

## Aggregate summaries

Three summary structures accelerate common queries without scanning
packed segments:

### Type counts

Per-database, per-type relationship counts.  Answers
`MATCH ()-[r:TYPE]->() RETURN count(r)` in O(1).

### Per-property sums

Per-database, per-type, per-property numeric sums.  Answers
`MATCH ()-[r:TYPE]->() RETURN sum(r.prop)` in O(1).

### Endpoint-pair tables

Per-database, per-type, per-(start-id, end-id) aggregate tables storing
`~count` plus numeric property sums.  These power labeled relationship
counts and sums:

```cypher
MATCH (:Zone)-[r:TRIP]->(:Zone)
RETURN count(r), sum(r.distance)
-- answered from the endpoint table without decoding any records
```

All three are persisted as versioned s-expression files alongside the
cache and restored on startup.  Any mutation clears them; they are rebuilt
from the in-memory index during the next query or restart.

## Write path

### Batch ingest

Bulk writes use `with-s3-batch`, which queues records into memory and
flushes them as packed segments of up to 20 000 records per S3 object.
Each batch is a single codec-encoded list containing the magic string
`"scalaxy-s3-batch-v1"` followed by `(op key value)` triples.

### Single-object writes

Non-batch writes go through `%s3-put-raw`, which uploads one ordinary
object per key.  A tombstone marker is also written to prevent older
batch segments from resurrecting deleted keys after restart.

### Immediate delete visibility

Deletes remove the key from the in-memory index immediately, so
subsequent queries observe the deletion without waiting for a restart.
A tombstone marker ensures the key stays deleted across rebuilds.

### Targeted invalidation

Node-property mutations preserve relationship aggregate caches.  Only
mutations whose keys have an `r:` local part clear the per-endpoint
tables; type-counts, sums, and label-ID summaries remain valid.

## Encryption at rest

When `SCALAXY_S3_ENCRYPTION_KEY` is set, every object body is encrypted
before upload using authenticated HMAC-SHA256 encryption:

```
format: SCX1 || nonce(16) || ciphertext || HMAC-SHA256 tag(32)
keystream: HMAC-SHA256(key, nonce ‖ counter-block) in CTR mode
authentication: HMAC-SHA256 over SCX1 ‖ nonce ‖ ciphertext
```

Decryption happens transparently after download, before the data enters
the local persistent cache (which stores plaintext on the trusted node).
Objects without the SCX1 magic prefix pass through unchanged, enabling
gradual migration of existing unencrypted data.

## Listing & pagination

Object listing uses **marker-based pagination** (`start-after` parameter
for ListObjectsV2) instead of opaque continuation tokens.  Query
parameters are emitted in **sorted order** as required by SigV4 canonical
request construction.

## Ownership counting

In a multi-node deployment each node holds both its primary keys and
replica copies.  Aggregate summaries (type counts, sums, endpoint pairs)
are accumulated per-node over locally held records.  The gateway fans out
aggregate requests to every peer and divides by the replication breadth
to produce cluster-wide unique counts.

For RF = 2 (primary + one synchronous replica), dividing by two yields
the correct unique count when replication is symmetric.

## Re-home

The `/api/rehome` admin endpoint moves keys that a node holds but does
not own (per the consistent-hash ring) to their owning peer.  With
`keep: true` the local copy is retained (presence repair); without it
the local copy is removed after acknowledged delivery.

```sh
curl -X POST http://node:8080/api/rehome \
  -H 'Content-Type: application/json' \
  -d '{"limit": 50000, "keep": true}'
```

Per-key error handling ensures one bad key doesn't abort the entire
repair; undeliverable keys are skipped and counted.

## Configuration

| Variable | Default | Description |
|---|---|---|
| `SCALAXY_STORE_BACKEND` | *(none)* | Set to `s3` to enable the S3 backend |
| `SCALAXY_S3_ENDPOINT` | *(none)* | S3-compatible endpoint URL |
| `SCALAXY_S3_BUCKET` | *(none)* | Per-node bucket name |
| `SCALAXY_S3_ACCESS_KEY` | *(none)* | S3 access key |
| `SCALAXY_S3_SECRET_KEY` | *(none)* | S3 secret key |
| `SCALAXY_S3_PREFIX` | `scalaxy/` | Key prefix (node ID appended automatically) |
| `SCALAXY_S3_LAZY` | `false` | Enable lazy loading (recommended for large datasets) |
| `SCALAXY_S3_ENCRYPTION_KEY` | *(none)* | Encrypt all S3 objects at rest |
| `SCALAXY_S3_CACHE_MAX_BYTES` | unlimited | Local cache eviction budget |
