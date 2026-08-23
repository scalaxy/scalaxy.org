---
date: 2026-08-21
lastmod: 2026-08-21
title: "S3 object storage"
description: "How Scalaxy stores graph data in S3-compatible object stores: packed immutable segments, lazy loading, local caching, aggregate summaries, encryption at rest, ownership counting, presence repair, and operational guidance."
weight: 32
---

Scalaxy's S3 storage backend replaces the local append-only log with an
S3-compatible object store for datasets that exceed local disk capacity or
require shared storage across a cluster. It supports Garage, MinIO,
AWS S3, Cloudflare R2, and any other S3-compatible API.

The backend is designed around three principles:

1. **Objects are immutable.** Data is written as packed, append-only
   segments. Deletes are tombstone markers. This means no partial writes
   and no read-modify-write races on individual objects.

2. **The local cache is the working set.** Every fetched S3 object is
   cached on local disk under its SHA-256 digest. After warm-up the cache
   hit rate reaches 100 %, so steady-state reads never touch the network.

3. **Summaries answer queries without decoding records.** Type counts,
   per-property sums, label-ID sets, top-k lists, and endpoint-pair
   tables are maintained per-node so that count, sum, avg, and labeled
   relationship queries are answered from in-memory hash tables in
   microseconds — even on graphs with millions of relationships.

---

## How data is organised

Every node maps to its own bucket and key prefix. Within that namespace,
objects fall into four categories:

### Packed batch segments

Bulk writes are grouped into immutable batches of up to 20 000 records.
Each segment is a single S3 object containing a codec-encoded list of
`(op key value)` triples with a `"scalaxy-s3-batch-v1"` magic string.
Segment keys use hex-encoded `@batch:<timestamp>-<seq>` names. Because
segments are write-once, there are no partial writes or read-modify-write
races.

When multiple segments contain the same key, the **last writer wins**
during replay. The replay engine tracks each key's final `(segment,
offset)` pair; only that entry is visible.

### Ordinary objects

Individual key/value pairs written outside batch mode — typically for
low-volume updates. Object keys use the same hex encoding as segments.

### Tombstone markers

Deletion markers stored as zero-byte objects keyed as
`@tombstone:<hex(key)>:<timestamp>`. Tombstones prevent older batch
segments from resurrecting deleted keys after restart.

### Sidecar indexes

Each segment has a companion `.idx` file with per-record metadata:
key, relative segment name, byte range, operation type. Plus a header
with version, count, key-range, and a 256-bit Bloom filter over all
keys. Sidecars are validated on load and rebuilt if corrupt.

---

## Lazy loading

Nodes boot without materialising the full key space. An **in-memory
index** maps each logical key to its cached segment and byte range.
Individual values are read on demand via range reads from the local
persistent cache.

Startup sequence:

1. LIST objects under prefix (marker-paginated, sorted params)
2. Classify: batch segments / tombstones / ordinary objects
3. Load ordinary objects into table
4. Replay segments via sidecars (or parse if no sidecar)
5. Process tombstones
6. Build endpoint-aggregate tables from owned-held records
7. Restore or rebuild summaries → mark valid

A 5.7 M-entry index is ready in ~4 minutes after cold start.

---

## Local persistent cache

Every fetched S3 object is cached on local disk under its SHA-256
digest. After warm-up the hit rate reaches 100 %. Cache integrity is
validated by checking the magic header; corrupt entries are deleted
and re-fetched. Eviction uses a configurable budget with oldest-first
policy.

---

## Aggregate summaries

Three summary structures accelerate graph queries without decoding
packed records:

| Summary | Answers | Example |
|---|---|---|
| Type counts | `count(r)` per type | `MATCH ()-[r:TRIP]->() RETURN count(r)` |
| Per-property sums | `sum(r.prop)` per type | `sum(r.distance)` |
| Endpoint pairs | Labeled count/sum per zone pair | `MATCH (:Zone)-[r:TRIP]->(:Zone) RETURN count(r), sum(r.distance)` |

All are persisted as versioned files and restored on startup.
Targeted invalidation preserves relationship summaries when only node
properties are mutated.

---

## Write path

Two modes: batch ingest (up to 20k records per object) and single-object
writes. Deletes are immediately visible: the key is removed from the
in-memory index before the tombstone is persisted.


---

## Encryption at rest

When `SCALAXY_S3_ENCRYPTION_KEY` is set, every object body is encrypted
before upload using authenticated HMAC-SHA256 encryption:

```
format: SCX1 || nonce(16) || ciphertext || HMAC-SHA256 tag(32)
keystream: HMAC-SHA256(key, nonce ‖ counter-block) in CTR mode
authentication: HMAC-SHA256 over SCX1 ‖ nonce ‖ ciphertext
```

Decryption happens transparently after download, before the data enters
the local persistent cache. Objects without the SCX1 magic prefix pass
through unchanged, enabling gradual migration of existing unencrypted data.

---

## Listing & pagination

Object listing uses **marker-based pagination** (`start-after` parameter
for ListObjectsV2) instead of opaque continuation tokens. Query
parameters are emitted in **sorted order** (`list-type`, `max-keys`,
`prefix`, then optionally `start-after`) as required by SigV4 canonical
request construction.

Marker-based pagination avoids two issues with continuation tokens:

1. Tokens are opaque binary blobs whose encoding may interact badly
   with URL-encoding and SigV4 signature computation.
2. Some S3 implementations return tokens that change format as the
   key space evolves, breaking forward compatibility.

---

## Ownership counting

In a multi-node deployment each node holds both its primary keys and
replica copies. Aggregate summaries — type counts, sums, endpoint pairs —
are accumulated per-node over **locally held records**.

The gateway fans out aggregate requests to every peer and sums their
responses. Because each live key exists on exactly RF nodes (primary +
one synchronous replica), dividing the raw sum by the replication
breadth produces the cluster-wide unique count:

```
raw_sum = Σ(local_count[node] for all nodes)
unique  = raw_sum / RF        -- when replication is symmetric
```

For the NYC taxi dataset on a 3-node cluster with RF = 2:
raw_sum ≈ 5 866 194, ÷2 → 2 933 097 unique trips ✓

### When ÷N breaks

The ÷N shortcut assumes uniform replication: every live key appears on
exactly N nodes. This invariant can break when:

- A node crashes during ingestion and some writes only reach one node.
- A delete propagates to some replicas but not others.
- Nodes rejoin with stale data that wasn't caught up.
- Keys are written directly to a non-owner during a partition.

When replication is non-uniform, ownership-based counting is needed:
each node counts only keys whose ring owner matches itself. This requires
the ring to match the actual data placement — which in turn requires
presence repair after any topology change.

Scalaxy currently uses the ÷N approach because it is simple and correct
when replication is symmetric. Ownership-based counting infrastructure
(ring on S3 config, per-key ownership checks) is implemented but not yet
enabled for production use due to placement mismatches from historical
crash damage.

---

## Presence repair

After node crashes or topology changes, some keys may end up held by
non-owner nodes. The `/api/rehome` endpoint delivers these keys to their
ring owners:

```sh
# Copy displaced keys to owners (presence repair; local copies retained)
curl -X POST http://node:8080/api/rehome \
  -H 'Content-Type: application/json' \
  -d '{"limit": 50000, "keep": true}'

# Move displaced keys to owners (remove local copies after delivery)
curl -X POST http://node:8080/api/rehome \
  -H 'Content-Type: application/json' \
  -d '{"limit": 50000}'
```

| Parameter | Default | Description |
|---|---|---|
| `limit` | 1000 | Maximum keys to process per call |
| `keep` | false | Retain local copy after delivery |

Returns `{"moved": N, "skipped": M}`. Call repeatedly until `moved`
reaches 0 to complete the repair. Per-key error handling ensures one
bad key doesn't abort the entire operation.

### Repair modes

| Mode | Behavior | Use case |
|---|---|---|
| `keep: true` | Copy to owner, retain locally | Restore redundancy after node loss |
| `keep: false` | Copy to owner, remove local copy | Rebalance after adding nodes |

With `keep: true`, the key ends up on both nodes (owner + current holder),
preserving RF = 2. With `keep: false`, the key migrates to the owner only,
reducing to RF = 1 until the next replication cycle.

---

## Elasticity and scalability

### Scaling reads

Read scalability comes from three layers working together:

1. **Local persistent cache** eliminates network I/O after warm-up.
   The cache stores full object bodies under SHA-256 digests and serves
   them from the OS page cache at memory speed. After warm-up the hit
   rate reaches 100 %.

2. **Aggregate summaries** eliminate record decoding for common queries.
   Type counts answer unlabeled relationship counts without touching any
   packed segment; endpoint-pair tables answer labeled count/sum queries
   by iterating pre-aggregated zone-to-zone routes instead of individual
   trips. Both are served from in-memory hash tables.

3. **Gateway fan-out** distributes aggregate requests across all nodes
   simultaneously. Each node computes its owned-held contribution in
   parallel, and the gateway sums the responses.

Adding nodes increases the total cache capacity and distributes the
fan-out load. No rebalancing is required for read scaling.

### Scaling writes

Write throughput is bounded by S3 PUT latency (~5–20 ms per object) and
the batch size. Bulk ingestion uses packed batches of up to 20 000
records per object, so the effective throughput is limited by the total
bytes written, not the number of S3 API calls.

For the 2.93M-trip NYC dataset (~730 MB per node), bulk ingest completes
in ~20 minutes across a 3-node cluster with RF = 2.

Writes are acknowledged synchronously: the primary applies the mutation,
replicates to its configured follower, and only then acknowledges the
client. Failed replications enter a durable outbox that persists across
restarts; a background thread retries undelivered messages every 5
seconds.

### Adding nodes

New nodes join the ring and begin receiving writes for their hash range.
Existing keys are not automatically migrated — they stay on their original
nodes as replicas for the previous owner's range. Over time, as new data
is ingested through the normal write path (which routes to ring owners),
the distribution converges.

To accelerate convergence, run presence repair on existing nodes to move
displaced keys to their new owners.

### Removing nodes

Before removing a node, its keys must be migrated to remaining nodes.
The cluster API provides `cluster-remove-node` which re-homes keys before
removal. Without this step, keys held only by the removed node become
unavailable.

---

## Scenarios

### Scenario: loading the NYC taxi dataset

```sh
# 1. Start Garage and create buckets for each node
docker compose -f docker-compose.yml up -d garage
# ... create buckets int-0, int-1, int-2 and access keys ...

# 2. Start the 3-node Scalaxy cluster
export SCALAXY_STORE_BACKEND=s3
export SCALAXY_S3_ENDPOINT=http://garage:3900
export SCALAXY_S3_LAZY=true
# ... per-node bucket/keys/prefix ...

# 3. Ingest the dataset (owner-routed via consistent hashing)
sbcl --dynamic-space-size 8192 \
     --script scripts/run-benchmark-nyc.lisp per-trip "" 1
# ← writes 2,933,097 relationships as ~147 batch objects across 3 buckets

# 4. Verify
curl -X POST http://localhost:8080/api/cypher \
  -d '{"query":"MATCH ()-[r:TRIP]->() RETURN count(r) AS c"}'
# → {"rows":[[2933097]]}
```

During ingest, each node builds its own index and summary files locally.
On subsequent restarts, these are loaded directly — no re-listing or
re-decoding needed.

### Scenario: point-in-time recovery

S3 objects are immutable and versioned (if bucket versioning is enabled).
To recover to a point in time:

1. List object versions with timestamps
2. Identify the last batch segment before the target time
3. Replay segments up to that point into a fresh store

This gives you point-in-time recovery without WAL shipping or binlogs.

### Scenario: read-only analytics replica

Because S3 objects are shared storage, a read-only analytics process can
mount the same bucket with a different prefix and query the data without
going through the write path:

```
analytics-node: prefix = "live-import-20260819node-0/"
                → sees exactly what node-0 sees
                → can run Cypher queries against the same data
```

No separate export/import step is needed.

### Scenario: encrypting an existing deployment

Encryption is backward compatible. To add encryption to an existing
deployment:

1. Set `SCALAXY_S3_ENCRYPTION_KEY` on all nodes
2. Restart nodes — existing unencrypted objects pass through transparently
3. New writes are encrypted automatically
4. Old unencrypted data migrates gradually as records are overwritten

No bulk re-encryption is required; the transition is gradual and safe.

---

## Performance tuning

### Cache budget

The default cache grows without bound. For dedicated nodes with
sufficient disk, this is recommended — it maximizes the hit rate.

For shared nodes, set `SCALAXY_S3_CACHE_MAX_BYTES` to limit disk usage.
Oldest-first eviction makes room for new entries. Note that evicted
segments must be re-fetched from S3 on next access.

### Dynamic space size

The SBCL heap (`--dynamic-space-size`) must be large enough for the
in-memory lazy-index (5.7M entries ≈ 2 GB) plus aggregate summaries.
8 GB is sufficient for the full NYC taxi dataset.

### Summary rebuild frequency

Summaries are invalidated on relationship mutations and rebuilt lazily
on the next query. For write-heavy workloads, consider batching writes
(`with-s3-batch`) so that invalidation happens once per batch rather
than once per individual write. Targeted invalidation preserves caches
on node-property mutations.

### Encryption overhead

Authenticated encryption adds ~10–15 % to write latency (HMAC-SHA256
keystream generation + authentication tag computation). Read latency is
less affected because decryption reuses the same keystream. For maximum
throughput, disable encryption during bulk ingestion and enable it
afterward (existing unencrypted objects pass through transparently).

### Batch size tuning

Batch segments pack up to 20 000 records per S3 object. Smaller batches
mean more S3 objects (slower listing) but faster per-batch flushes.
Larger batches mean fewer objects but more data loss risk if a batch
write fails. The default of 20 000 balances both concerns for most
workloads.

---

## Formal specification

Scalaxy's storage semantics are formally specified in TLA+
(`specs/tla/ScalaxySpec.tla`). This specification serves as the central
truth for correctness guarantees:

| Invariant | Guarantee |
|---|---|
| NoDataLoss | Every written key that hasn't been deleted is readable from some up node |
| DeleteVisible | Tombstoned keys never appear as live data |
| OwnershipConsistency | Each key has at least one node holding it |
| EncryptionConsistency | Encryption state is consistent across all nodes |

The specification models a 2-node cluster with 2 keys — small enough for
exhaustive model checking via TLC while capturing the essential correctness
properties of the distributed storage layer.

---
