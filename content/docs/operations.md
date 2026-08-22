---
date: 2026-08-06
lastmod: 2026-08-21
title: "Operations"
description: "Health checks, failure behavior, durability, encryption at rest, re-homing, and troubleshooting."
weight: 90
---

## Monitoring

- **Health**: poll `/healthz`; non-200 means the node is not serving.
- **Status**: `/api/status` returns cluster state: `healthy`, `degraded`, or node-level `unreachable`.
- **Metrics**: per-node key counts and uptime via `/api/node-status` (used by the dashboard's cluster view).
- **S3 cache**: per-node hit/miss counters and bytes read via `/api/status` → `s3Cache`.
- **Aggregate summaries**: `s3SummaryValid` shows whether cached aggregate summaries are current.

## Failure behavior

| Event | What happens |
|---|---|
| Node crash | Durability log replayed on restart; acknowledged writes survive. S3 objects are authoritative. |
| Node unreachable | Reads fail over to replicas; writes go to a reachable member; status shows `degraded`. |
| Node rejoin | Ownership resumes; status returns to `healthy`. Durable outbox delivers missed replications. |
| Corrupt cache entry | Detected by magic-header validation; silently deleted and re-fetched from S3. |

## Encryption at rest

Set `SCALAXY_S3_ENCRYPTION_KEY` to enable authenticated encryption of all
data stored in S3. Each object body is encrypted with HMAC-SHA256 CTR-mode
encryption before upload:

```
format: SCX1 || nonce(16) || ciphertext || HMAC-SHA256 tag(32)
```

Decryption is transparent on read. Objects without the SCX1 magic prefix
(legacy unencrypted data) pass through unchanged.

The local persistent cache stores decrypted data for performance; it is
on the trusted node's local disk and does not need encryption.

## Re-homing displaced keys

When nodes crash or rejoin, some keys may end up held by non-owner nodes.
The `/api/rehome` endpoint delivers these keys to their ring owners:

```sh
# Copy displaced keys to owners (presence repair)
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
reaches 0 to complete the repair. With `keep: true` the local copy is
retained as a replica (RF = 2 is preserved).

Per-key error handling ensures one bad key doesn't abort the entire
repair operation.

## Aggregate summaries

Three summary structures accelerate common queries:

| Summary | Answers | Invalidated by |
|---|---|---|
| Type counts | `count(r)` per type | Relationship mutations |
| Per-property sums | `sum(r.prop)` per type | Relationship mutations |
| Endpoint-pair tables | Labeled count/sum per (start,end) pair | Relationship mutations |
| Label-ID sets | Zone/node visibility in labeled queries | Node label changes |

All are rebuilt from the in-memory index during the next query or
restart. Targeted invalidation preserves relationship summaries when
only node properties are mutated.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Queries return 0 after restart | Summaries not yet loaded | Wait for `s3SummaryValid` to become true |
| Zone count lower than expected | Displaced zones not visible at owners | Run presence repair (`/api/rehome {"keep":true}`) |
| Count differs across restarts | Non-uniform replication topology | Verify all nodes hold their primary keys; run presence repair |
| Cache miss rate high | Local cache was wiped | Normal — cache refills automatically from S3 |
| `SCX1` errors on read | Wrong decryption key | Verify `SCALAXY_S3_ENCRYPTION_KEY` matches the key used for writing |
