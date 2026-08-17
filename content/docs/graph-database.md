---
date: 2026-08-17
lastmod: 2026-08-17
title: "Graph database"
description: "The property-graph layer: nodes, relationships, labels, indexes, and multi-database support over the replicated store."
weight: 25
---

Scalaxy's graph layer turns every database into a **property graph** on
top of the same replicated, durable key/value store.  Graph entities share
the ring, the durability log, and the failover story with keys — nothing
extra to deploy.

## Data model

- **Node** — an id, a set of labels, and a property map.  A node can carry
  any number of labels (`:Person`, `:Movie`, ...).
- **Relationship** — an id, a type, a start node, an end node, and a
  property map.  Relationships are always directed; queries can match them
  in either direction.
- **Property values** — scalars, strings, booleans, lists, and maps; large
  binary properties spill to blob records so the graph stays cheap to
  scan.

## Storage

- **Over the KV store** — nodes, relationships, and the index structures
  are ordinary records in the replicated store (`src/graph.lisp`,
  `src/db.lisp`, `src/codec.lisp`), so they are sharded by consistent
  hashing, appended to the durability log, and replayed on startup like
  any other mutation.
- **Indexes** — label, type, and adjacency (outgoing/incoming) indexes
  keep `MATCH (a:Label)-[:TYPE]->(b)` local to the relevant entity sets.
- **Id minting** — entity ids are minted by the database layer, with
  binary codec support for round-tripping entities through the wire
  protocol and the log.

## Multi-database

A database is a namespaced partition of the same ring.  Keys, graph
entities, and logs live per database; the default database is `default`.
Create and drop databases through the client API, and target any database
from a Cypher query via the `db` parameter:

```sh
curl -X POST http://localhost:8080/api/cypher \
  -d '{"query":"MATCH (n) RETURN count(*) AS n","db":"analytics"}'
```

## From Lisp

```lisp
(scalaxy:cypher *db* "CREATE (m:Movie {title: 'The Matrix', released: 1999}) RETURN m.title")
(scalaxy:cypher *db* "MATCH (m:Movie) WHERE m.released > 2000 RETURN m.title ORDER BY m.title")
```

## Next

- [Cypher](/docs/cypher/) — the query language, conformance results, and reference.
- [Benchmarks](/docs/benchmarks/) — the Movie Graph and NYC taxi datasets.
