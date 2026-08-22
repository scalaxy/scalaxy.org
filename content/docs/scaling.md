---
date: 2026-08-21
lastmod: 2026-08-21
title: "Scaling & elasticity"
description: "How Scalaxy scales with your data: adding nodes, presence repair, aggregate summaries, read fan-out, and operational guidance for growing deployments."
weight: 33
---

## How Scalaxy scales

Scalaxy uses **consistent hashing with virtual nodes** (128 per physical
node) to distribute keys across a cluster. Adding or removing a node
moves only ~1/N of the keys. Every node can serve reads for the entire
dataset because each key is replicated to at least one other node.

When backed by S3 object storage, each node also maintains:

- A **local persistent cache** of fetched objects (grows to hold the
  full working set)
- An **in-memory lazy index** mapping keys to cached segment locations
- **Aggregate summaries** (type counts, sums, endpoint pairs) built
  during startup and invalidated on relationship mutations

These per-node structures mean that query latency stays flat as the
cluster grows: each node serves its owned-held contribution from local
resources.

## Adding nodes

New nodes join the ring and begin receiving writes for their hash range.
Existing data is not automatically migrated — it stays on the original
nodes as replicas. Over time, new writes converge to the correct owners.

To accelerate convergence, run presence repair on existing nodes after
the new node is stable:

```sh
# On each existing node:
curl -X POST http://node:8080/api/rehome \
  -H 'Content-Type: application/json' \
  -d '{"limit": 50000}'
```

This delivers misowned keys to their ring owners. Repeat until `moved`
reaches 0.

## Removing nodes

Before removing a node, ensure its keys are present on other nodes.
The cluster API provides `cluster-remove-node` which re-homes keys
before removal completes. Without this step, keys held only by that
node become unavailable.

## Presence repair

After crashes or topology changes, some keys may be held by non-owner
nodes. The `/api/rehome` endpoint copies these to their ring owners:

```sh
curl -X POST http://node:8080/api/rehome \
  -d '{"limit": 50000, "keep": true}'
```

With `keep: true`, both nodes hold the key (RF = 2 preserved).
Repeat until no more keys need delivery. The operation makes
incremental progress — you can run it in batches without downtime.

## Aggregate summaries and scaling

Each node maintains independent summary tables over its locally held
records:

| Summary | Per-node | Gateway behavior |
|---|---|---|
| Type counts | Owned-held trip count | Sum across peers |
| Property sums | Owned-held sum per property | Sum across peers |
| Endpoint pairs | Owned-held count/sum per zone pair | Sum across peers |
| Label-ID sets | Owned-held zone/node IDs | Union across peers |

The gateway fans out aggregate requests to all peers simultaneously.
Each node computes its owned-held contribution from in-memory hash
tables — no record decoding needed. The gateway divides by the
replication breadth to produce cluster-wide unique aggregates.

### Scaling aggregate performance

Aggregate query latency stays flat as the dataset grows because:

1. Each node iterates only its **owned** endpoint pairs (not all records)
2. Pairs are pre-aggregated per zone-to-zone route (~69k pairs vs 2.9M trips)
3. All nodes compute in parallel

For the NYC taxi dataset on a 3-node cluster, labeled count queries
return in **single-digit milliseconds** regardless of whether there are
100k or 2.9M relationships.

## Read fan-out

Cypher queries flow through the gateway to individual nodes via TCP:

```
client → gateway (node-0) → op-aggregate → node-0, node-1, node-2
                              ↓ each computes locally
                           ← responses summed by gateway
                           → result returned to client
```

Each peer processes independently. No coordination protocol is needed
for reads — every member can compute its owned-held contribution from
its own index and summaries.

## Monitoring scaling

Key metrics to watch as the cluster grows:

| Metric | Where | Healthy value |
|---|---|---|
| Cache hit rate | `/api/status` → `s3Cache.hits / (hits+misses)` | > 99 % after warm-up |
| Summary valid | `/api/status` → `s3SummaryValid` | true |
| Keys per node | `/api/status` → `keys` | roughly equal across nodes |
| Node status | `/api/status` → `status` | "ok" |

If cache hit rate drops below 99 %, check disk usage on the affected
node — the persistent cache may have been evicted or wiped.
