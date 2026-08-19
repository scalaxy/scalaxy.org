---
date: 2026-08-06
lastmod: 2026-08-17
title: "Getting started"
description: "Install SBCL, clone Scalaxy, run the tests, and start your first node."
weight: 10
---

## Requirements

- [SBCL](https://www.sbcl.org/) 2.x — the reference implementation.
- Git, Make, and a POSIX shell for the launcher scripts.
- No external Lisp libraries are required: the JSON, HTTP, and TCP layers are implemented in-tree.

## Get the code

```sh
git clone https://github.com/scalaxy/scalaxy.git
cd scalaxy
```

## Run the test suite

```sh
make test
```

This runs **9,018 checks** across 33 groups: consistent hashing, durable storage, protocol round-trips, replication, cluster churn, real TCP traffic, JSON, HTTP, the web API, gateway routing, status aggregation, node-failure failover, the property-graph layer, the openCypher engine (lexer, parser, AST round-trips, executor, semantics, updates, TCP/gateway/web integration, and a differential harness against a reference oracle), multi-database support, binary codec, and blob spills.

A separate **conformance suite** runs the openCypher TCK: all 3,898 expanded scenarios currently pass with zero failures or unsupported cases. See [Cypher conformance](/docs/cypher/).

## Start a node

```sh
bin/scalaxy --address 127.0.0.1:7200 --http-address 127.0.0.1:8080 --data-dir ./data
```

You should see:

```
Scalaxy node <hostname>
  data:  127.0.0.1:7200
  http:  127.0.0.1:8080 (web console)
```

## Use the web console

Open <http://127.0.0.1:8080> — the dashboard shows cluster status, keys, the ring distribution, and a command console:

![Scalaxy web console — cluster overview](/img/screenshots/scalaxy1.jpg)

![Scalaxy web console — data browser](/img/screenshots/scalaxy2.jpg)

Or use the REST API directly:

```sh
curl -X PUT http://127.0.0.1:8080/api/keys/greeting -d '{"value":"hello world"}'
curl http://127.0.0.1:8080/api/keys/greeting
# => {"key":"greeting","size":11,"utf8":"hello world","hex":"68656C6C6F20776F726C64"}
```

## Query the graph

Every database also holds a property graph, queried with openCypher through
the same node:

```sh
curl -X POST http://127.0.0.1:8080/api/cypher \
  -d '{"query":"CREATE (p:Person {name: \"Alice\"})-[:KNOWS]->(q:Person {name: \"Bob\"}) RETURN p.name AS from, q.name AS to"}'
# => {"columns":["from","to"],"rows":[["Alice","Bob"]],"count":1}

curl -X POST http://127.0.0.1:8080/api/cypher \
  -d '{"query":"MATCH (a:Person)-[:KNOWS]->(b:Person) RETURN a.name, b.name"}'
```

The language covers `MATCH` / `OPTIONAL MATCH` / `WHERE` / `WITH` /
`RETURN` / `UNWIND` / `ORDER BY` / `SKIP` / `LIMIT` / `DISTINCT` /
`UNION`, `CREATE` / `MERGE` / `SET` / `REMOVE` / `DELETE` /
`DETACH DELETE`, named paths, variable-length relationships, and full
aggregation.  See [Graph database](/docs/graph-database/) and
[Cypher](/docs/cypher/).

## From a Lisp REPL

```lisp
(require :asdf)
(asdf:load-asd "/path/to/scalaxy/scalaxy.asd")
(asdf:load-system "scalaxy")

(defvar *db* (scalaxy:connect :host "127.0.0.1" :port 7200))
(scalaxy:put *db* "answer" "42")
(scalaxy:octets-to-string (scalaxy:get *db* "answer")) ; => "42"
```

## Next steps

- [Architecture](/docs/architecture/) — how the pieces fit together.
- [Deployment](/docs/deployment/) — Docker, compose, and configuration.
- [Kubernetes](/docs/kubernetes/) — run a replicated cluster on Kubernetes.
