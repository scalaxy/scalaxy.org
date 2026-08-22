---
date: 2026-08-17
lastmod: 2026-08-21
title: "Graph database"
description: "The property-graph layer: how nodes, relationships, and aggregate summaries work over S3-backed object storage. Includes lazy loading, ownership counting, and multi-database support."
weight: 25
---

Scalaxy's graph layer turns every database into a **property graph**
backed by S3-compatible object storage. Graph entities share the same
storage backend, ownership model, and durability guarantees as key/value
records — there is no separate graph service to deploy.

## Data model

- **Node**: an id, a set of labels, and a property map. A node can carry
  any number of labels (`:Person`, `:Zone`, `:Movie`).
- **Relationship**: an id, a type, a start node, an end node, and a
  property map. Relationships are always directed; queries can match
  them in either direction.
- **Property values**: scalars, strings, booleans, lists, maps, and
  temporal values. Large binary properties spill to blob records.

## How graph data is stored

Graph entities are ordinary records in the S3-backed store. Each entity
is encoded with a type-tagged binary codec and stored as either:

- A **packed batch segment** (during bulk ingestion, up to 20 000 records
  per S3 object), or
- An **ordinary object** (for individual writes outside batch mode).

Every record has a structured key that encodes the database name, the
entity kind, and the entity id:

```
d:<database>:n:<node-id>     → node record
d:<database>:r:<rel-id>      → relationship record
d:<database>:e:<start>:<type>:<end>  → endpoint index entry
```

This means graph data benefits from all the S3 storage features: packed
segments, lazy loading, local caching, aggregate summaries, encryption
at rest, and marker-based pagination.

### Lazy index

On startup, each node builds an in-memory **lazy index** mapping every
graph key to its location within a cached segment. Individual values
are read on demand via byte-range reads from the local persistent
cache — no full segment decode needed.

For the NYC taxi dataset, this index contains 5.7 million entries
across 3 nodes and is ready in ~4 minutes after a cold start.

## Aggregate summaries

Three summary structures accelerate common graph queries without
decoding any packed segment records:

| Summary | Answers | Complexity |
|---|---|---|
| Type counts | `count(r)` per type | O(1) |
| Per-property sums | `sum(r.prop)` per type | O(1) |
| Endpoint-pair tables | Labeled count/sum per zone pair | O(pairs) |

These are maintained per-node during segment replay and persisted as
versioned files. After a restart, they are loaded from disk — no
re-scan needed.

### Endpoint-pair tables

For labeled relationship queries like
`MATCH (:Zone)-[r:TRIP]->(:Zone) RETURN count(r)`, the engine needs to
count trips where both endpoints are Zone nodes. The endpoint-pair
table pre-aggregates this per zone pair:

```
("taxi-live-full", "TRIP", "g000001", "g000002") → {
    "~count": 1523,
    "distance": 5491.25,
    "fare": 23418.75
}
```

At query time, the gateway resolves the Zone ID set from label-ID
summaries, fans out to all nodes, and each node sums its owned-held
endpoint pairs whose start and end are both in the resolved set.

This means a query over 2.9M relationships is answered by iterating
~69k pre-aggregated zone-pair entries instead of scanning 2.9M records.

### Targeted invalidation

Not all mutations affect all summaries. Node-property updates (which
change no relationship counts, sums, or endpoint pairs) preserve
relationship aggregate caches. Only mutations whose keys have an `r:`
local part clear the per-endpoint tables and type-counts.

This means mixed workloads — updating user profiles while running
trip analytics — don't degrade each other's query performance.

## Query flow

Here's how a labeled aggregate query flows through the system:

```
1. Cypher: MATCH (:Zone)-[r:TRIP]->(:Zone) RETURN count(r)

2. Gateway: parse query → detect aggregate pushdown opportunity
   → resolve Zone IDs from label-ID summaries (merged across nodes)
   → send op-aggregate to all peers with function=COUNT,
     type=TRIP, left-label=Zone, right-label=Zone

3. Each peer node:
   a. Check per-shape aggregate cache → return cached result if hit
   b. Check endpoint-pair fast path → iterate pre-aggregated pairs
   c. Fall back to filtered scan over lazy-index (slow path)

4. Gateway: sum peer responses, divide by replication breadth
   → return cluster-wide unique count
```

Steps 3a and 3b complete in **single-digit milliseconds** for the
NYC taxi dataset. Step 3c (full scan) takes ~30 seconds but is only
needed after mutations invalidate the summaries.

## Multi-database

A database is a namespaced partition of the same ring. Keys, graph
entities, and summaries live per database; the default database is
`default`. Create and drop databases through the client API, and
target any database from a Cypher query via the `db` parameter.

## From Lisp

```lisp
(scalaxy:cypher *db* "CREATE (m:Movie {title: 'The Matrix', released: 1999}) RETURN m.title")
(scalaxy:cypher *db* "MATCH (m:Movie) WHERE m.released > 2000 RETURN m.title ORDER BY m.title")
```

## Next

- [S3 object storage](/docs/s3-storage/): the storage backend that
  powers the graph layer.
- [Cypher](/docs/cypher/): the query language, conformance results, and reference.
- [Benchmarks](/docs/benchmarks/): the Movie Graph and NYC taxi datasets.
