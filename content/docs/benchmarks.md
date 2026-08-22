---
date: 2026-08-17
lastmod: 2026-08-21
title: "Benchmarks"
description: "Two shipped benchmark datasets (the Neo4j Movie Graph and the NYC taxi graph), with provenance and reference timings."
weight: 55
---

Two benchmark datasets ship with the repository (`benchmarks/`), covering
the engine at two very different scales: a small curated graph and a
real-world **large** graph.

## Movie Graph

The canonical Neo4j Movie Graph tutorial dataset:

| Metric | Value |
|---|---|
| Nodes / relationships | 171 / 253 |
| Labels | `Movie` (38), `Person` (133) |
| Relationship types | `ACTED_IN`, `DIRECTED`, `PRODUCED`, `WROTE`, `FOLLOWS`, `REVIEWED` |
| License | Apache-2.0 (neo4j-graph-examples) |

The 15-query suite covers counts, filters, projections, `ORDER BY`/`LIMIT`,
aggregation (`count`/`avg`/`collect`), paths, variable-length
relationships, and string predicates.

```sh
sbcl --script scripts/run-benchmark.lisp          # 20 iterations by default
```

Reference (Apple Silicon MacBook, SBCL): dataset loads in ~41 ms, and
the 15-query suite runs in ~0.06–3.2 ms per query (the slowest is
"actors who directed themselves").

## NYC taxi graph

A **large-scale** graph built from real NYC TLC taxi-trip records: 263
taxi-zone nodes and `[:TRIP]` relationships in two modes:

| Mode | Relationships | Notes |
|---|---|---|
| `aggregated` | 25,711 | one edge per zone pair, with `trips`/`distance`/`fare`/`passengers` sums |
| `per-trip` | 2,933,097 | one edge per trip (Jan 2024 yellow taxi) |

The per-trip graph loads in ~20 s with an 8 GB heap. This measures a
2.93M-edge load. A reproducible `prepare.py` pipeline downloads the TLC zone lookup
and trip parquet and regenerates the CSV inputs; the raw downloads and the
large per-trip CSV are git-ignored.

### S3-backed cluster benchmarks

The full per-trip graph (2,933,097 relationships) deployed on a
3-node Garage S3 cluster with lazy loading, local persistent caches,
and aggregate summaries:

| Query | Latency | Notes |
|---|---|---|
| Unlabeled count (`count(*)`) | 5 ms | served from type-count summary |
| Labeled count (Zone→Zone) | 6–80 ms | served from endpoint-pair tables |
| Labeled sum (distance) | 7–29 ms | served from per-property sums |
| Top-5 ORDER BY DESC | 3 ms | top-k summary |
| Zone count | 3 ms | label-ID summary |
| Cache hit rate (warm) | 100 % | zero S3 GETs after warm-up |
| Restart to first query | < 4 min | full index rebuild from sidecars + cache |

All timings measured on Apple Silicon MacBook with Docker containers.
Cache is fully warm after the initial load; steady-state queries never
touch the network.

### Write-path characteristics

Single-object creates are acknowledged synchronously with replication.
Deletes are immediately visible without restart. Bulk ingestion uses
packed batches of up to 20 000 records per S3 object.

After any write, aggregate summaries are invalidated for relationship
mutations (targeted invalidation preserves caches on node-property
updates). The next read rebuilds affected summaries from the in-memory
index — a one-time cost per write batch, not per query.

Full provenance, license notes, file layout, and timings are in
`benchmarks/movies/README.md` and `benchmarks/nyc-taxi/README.md`.
