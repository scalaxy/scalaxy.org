---
date: 2026-08-06
lastmod: 2026-08-06
title: "Client API"
description: "Talk to Scalaxy from Lisp, curl, or your own language over the wire protocol."
weight: 60
---

## Lisp client

```lisp
(require :asdf)
(asdf:load-asd "/path/to/scalaxy/scalaxy.asd")
(asdf:load-system "scalaxy")

(defvar *db* (scalaxy:connect :host "127.0.0.1" :port 7200))

(scalaxy:put *db* "user:1" "Alice")                ; => T
(scalaxy:get *db* "user:1")                        ; octet vector
(scalaxy:octets-to-string (scalaxy:get *db* "user:1"))
;; => "Alice"

(scalaxy:scan *db* "user:")                        ; ((key . value) ...)
(scalaxy:delete *db* "user:1")                     ; => T
```

Values are octet vectors, so any binary payload stores as-is. The client speaks the wire protocol directly over TCP — no HTTP required.

## curl

The REST API ([reference](/docs/rest-api/)) is the easiest way to integrate from scripts:

```sh
curl -X PUT http://localhost:8080/api/keys/flag -d '{"value":"on"}'
curl http://localhost:8080/api/keys/flag
curl "http://localhost:8080/api/keys?prefix=flag"
curl -X DELETE http://localhost:8080/api/keys/flag
```

## Other languages

The [wire protocol](/docs/protocol/) is a length-prefixed binary format over TCP. Any language with TCP sockets can implement a client in a few hundred lines — see the opcode table for the exact layout.

## Cluster-aware routing

When a node runs with `SCALAXY_PEERS` set, REST and console operations route to ring owners automatically. For raw TCP clients, connect to any node and use the ring hashing rules to target the owner, or use the REST API as a gateway.

{{< callout type="info" >}}
The in-process `make-cluster` API embeds routing and replication in your own Lisp process — useful for tests and single-process services.
{{< /callout >}}
