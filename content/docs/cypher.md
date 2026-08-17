---
date: 2026-08-17
lastmod: 2026-08-17
title: "Cypher"
description: "The openCypher query language in Scalaxy: coverage, conformance certification, and access points."
weight: 35
---

Scalaxy implements the **openCypher** query language for its graph layer,
entirely in Common Lisp (SBCL) with zero external dependencies.  The
compiler pipeline lives in `src/cypher/`: lexer → parser → AST (with a
canonical printer) → semantic analysis → executor, plus a wire format and
a metacircular reference oracle used for differential testing.

## Language coverage

- **Clauses** — `MATCH`, `OPTIONAL MATCH`, `WHERE`, `WITH`, `RETURN`,
  `UNWIND`, `ORDER BY`, `SKIP`, `LIMIT`, `DISTINCT`, `UNION`, `CREATE`,
  `MERGE` (with `ON CREATE` / `ON MATCH`), `SET` (including map `=`/`+=`
  with null-removal), `REMOVE`, `DELETE`, `DETACH DELETE`.
- **Patterns** — labels, relationship types and direction, property
  constraints, bound-variable anchoring, named paths (`MATCH p = ...`),
  and variable-length relationships (`-[:T*1..3]->`) with
  relationship-list binding.
- **Expressions** — literals, maps, lists, list comprehensions, pattern
  comprehensions, list predicates (`all`/`any`/`none`/`single`),
  `EXISTS { }`, `CASE`, chained comparisons, list concatenation,
  append/prepend, and string/aggregation functions.
- **Aggregation** — symbolic aggregation, so aggregates can nest inside
  arbitrary expressions (`count(a) * 10 + count(b) * 5`,
  `head(collect(...))`), with implicit grouping, `DISTINCT` aggregates,
  and per-group `ORDER BY`.
- **Errors** — the openCypher error taxonomy as ~35 named CLOS
  conditions, surfaced through the REST API with `error` and `kind`
  fields.

## Conformance (openCypher TCK)

The engine is measured against the **official openCypher TCK** — 3,898
scenarios (outline expansion included) — via `scripts/run-tck.lisp`:

| Outcome | Scenarios |
|---|---|
| Pass | **2,726** |
| Fail (bugs in supported features) | 6 |
| Unsupported (declared out of scope) | 1,166 |

The unsupported bucket is a declared gap list, not hidden failure: 1,069
scenarios need temporal types (`date`/`time`/`datetime`/`duration`), 52
need stored procedures (`CALL`), 15 need full subquery update semantics
(`control query`), 13 need `percentileCont`/`percentileDisc`, and the rest
are small harness or error-kind corner cases (including six scenarios the
openCypher spec itself cannot satisfy under three-valued null semantics).
The full breakdown, the certified feature list, and the per-scenario log
are in the in-repository report: `docs/cypher-certification.md`.

The six remaining failures are narrow edge cases: three `RETURN`
column-name fidelity checks (openCypher keeps the literal source text of an
expression as its column name, e.g. `cOuNt( * )`), one variable-length
`count(p)` multiplicity nuance, one label-predicate column-name form, and
one `@skipGrammarCheck` map-key error-kind corner.

## Access points

- **REST** — `POST /api/cypher` (see [REST API](/docs/rest-api/)).
- **Lisp client** — `(scalaxy:cypher client query &key db params)` (see
  [Client API](/docs/client-api/)).
- **Wire protocol** — the `CYPHER` opcode (12) over the binary frames (see
  [Protocol](/docs/protocol/)).
- **Web console** — the `cypher <query>` command in the console's command
  bar.

## Reference

The complete language reference lives in the repository:
`docs/cypher-reference.md`.  For the design rationale, see
`docs/cypher-implementation-plan.md`.

## Next

- [Benchmarks](/docs/benchmarks/) — Movie Graph and NYC taxi datasets that
  exercise the engine at two very different scales.
