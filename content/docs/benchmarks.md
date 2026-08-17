---
date: 2026-08-17
lastmod: 2026-08-17
title: "Benchmarks"
description: "Two shipped benchmark datasets — the Neo4j Movie Graph and the NYC taxi graph — with provenance and reference timings."
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

The per-trip graph loads in ~20 s (8 GB heap) — a genuine large-database
test.  A reproducible `prepare.py` pipeline downloads the TLC zone lookup
and trip parquet and regenerates the CSV inputs; the raw downloads and the
large per-trip CSV are git-ignored.

```sh
sbcl --script scripts/run-benchmark-nyc.lisp aggregated "" 20    # 25,711 edges
sbcl --script scripts/run-benchmark-nyc.lisp per-trip 200000 5   # interactive size
sbcl --dynamic-space-size 8192 --script scripts/run-benchmark-nyc.lisp per-trip "" 1
```

Reference: aggregated-mode queries run in ~0.3–190 ms (a full-table
`count(*)` over 25,711 edges ~170 ms); per-trip queries over 100,000 edges
~0.5–0.8 s.  The suite surfaced a known scalability area: `MATCH`
materializes rows before aggregation, so whole-graph aggregations over the
full 2.93M-edge graph need a large heap (streaming aggregation is future
work).

Full provenance, license notes, file layout, and timings are in
`benchmarks/movies/README.md` and `benchmarks/nyc-taxi/README.md`.
