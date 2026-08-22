---
date: 2026-08-17
lastmod: 2026-08-18
title: "Cypher"
description: "The openCypher query language in Scalaxy: coverage, TCK conformance, and access points."
weight: 35
---

Scalaxy implements the **openCypher** query language for its graph layer,
implemented in Common Lisp (SBCL); the protocol, JSON, and HTTP layers are
in-tree. The compiler pipeline lives in {{< src "src/cypher/" >}}: lexer → parser → AST (with a
canonical printer) → semantic analysis → executor, plus a wire format and
a metacircular reference oracle used for differential testing.

## Language coverage

- **Clauses**: `MATCH`, `OPTIONAL MATCH`, `WHERE`, `WITH`, `RETURN`,
  `UNWIND`, `ORDER BY`, `SKIP`, `LIMIT`, `DISTINCT`, `UNION`, `CREATE`,
  `MERGE` (with `ON CREATE` / `ON MATCH`), `SET` (including map `=`/`+=`
  with null-removal), `REMOVE`, `DELETE`, `DETACH DELETE`.
- **Patterns**: labels, relationship types and direction, property
  constraints, bound-variable anchoring, named paths (`MATCH p = ...`),
  and variable-length relationships (`-[:T*1..3]->`) with
  relationship-list binding.
- **Expressions**: literals, maps, lists, list comprehensions, pattern
  comprehensions, list predicates (`all`/`any`/`none`/`single`),
  `EXISTS { }`, `CASE`, chained comparisons, list concatenation,
  append/prepend, and string/aggregation functions.
- **Aggregation**: symbolic aggregation, so aggregates can nest inside
  arbitrary expressions (`count(a) * 10 + count(b) * 5`,
  `head(collect(...))`), with implicit grouping, `DISTINCT` aggregates,
  and per-group `ORDER BY`.
- **Errors**: the openCypher error taxonomy as ~35 named CLOS
  conditions, surfaced through the REST API with `error` and `kind`
  fields.

## Conformance (openCypher TCK)

Scalaxy currently passes the complete **openCypher TCK**: all 3,898
expanded scenarios execute successfully with no failures or unsupported
cases. The local runner intentionally executes every corpus case, including
the scenario marked `@ignore` in the upstream feature file.

| Outcome | Scenarios |
|---|---:|
| Pass | **3,898** |
| Fail | **0** |
| Unsupported | **0** |

Run the same check from the repository root with:

```sh
sbcl --script scripts/run-tck.lisp
```

This is a reproducible TCK conformity result for the current Scalaxy
implementation; it is not a claim of third-party certification by the
openCypher project. The detailed runner and feature corpus are kept in the
repository alongside the implementation.

## Access points

- **REST**: `POST /api/cypher` (see [REST API](/docs/rest-api/)).
- **Lisp client**: `(scalaxy:cypher client query &key db params)` (see
  [Client API](/docs/client-api/)).
- **Wire protocol**: the `CYPHER` opcode (12) over the binary frames (see
  [Protocol](/docs/protocol/)).
- **Web console**: the `cypher <query>` command in the console's command
  bar.

## Reference

The complete language reference lives in the repository:
`docs/cypher-reference.md`.  For the design rationale, see
`docs/cypher-implementation-plan.md`.

## Next

- [Benchmarks](/docs/benchmarks/): Movie Graph and NYC taxi datasets that
  exercise the engine at two very different scales.
