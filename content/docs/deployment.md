---
date: 2026-08-06
lastmod: 2026-08-06
title: "Deployment"
description: "Docker, docker-compose, and the SCALAXY_* configuration reference."
weight: 70
---

## Configuration reference

Scalaxy is configured entirely through environment variables (or CLI flags).

| Variable | Default | Description |
|---|---|---|
| `SCALAXY_NODE_ID` | hostname | Ring member id |
| `SCALAXY_ADDRESS` | `0.0.0.0:7200` | Data-plane TCP listen address |
| `SCALAXY_HTTP_ADDRESS` | `0.0.0.0:8080` | Web console / REST / health address |
| `SCALAXY_DATA_DIR` | `./scalaxy-data` | Durability log directory |
| `SCALAXY_PEERS` | *(none)* | Cluster topology: `id=host:data-port[:http-port],...` |
| `SCALAXY_REPLICATE_TO` | *(none)* | Synchronous replication targets (same format) |
| `SCALAXY_WEB_DIR` | `web/` | Console assets location |

CLI equivalents: `--address`, `--http-address`, `--data-dir`, `--id`, `--peers`, `--replicate-to`, `--web-dir`.

## Docker

```sh
docker build -t scalaxy .
docker run --rm -p 8080:8080 -p 7200:7200 -v scalaxy-data:/var/lib/scalaxy scalaxy
# console: http://localhost:8080
```

The image is multi-stage: the builder compiles the systems and runs the full test suite at build time, so a broken build never produces an image. The runtime runs as a non-root user (uid 1000) with a `HEALTHCHECK` on `/healthz`.

## docker-compose (3-node cluster)

```sh
docker compose up --build
# console: http://localhost:8080
```

Each service sets `SCALAXY_PEERS` to the full topology and `SCALAXY_REPLICATE_TO` to its downstream replica target. Data lives in named volumes.

## Cluster configuration

```sh
export SCALAXY_PEERS="node-0=scalaxy-0:7200:8080,node-1=scalaxy-1:7200:8080,node-2=scalaxy-2:7200:8080"
export SCALAXY_REPLICATE_TO="node-1=scalaxy-1:7200"
bin/scalaxy
```

- `SCALAXY_PEERS` — the full ring membership. Every node should agree on this list so ownership is consistent.
- `SCALAXY_REPLICATE_TO` — where this node sends synchronous replicas. A common layout is a replication ring: node-0 → node-1 → node-2 → node-0.
- The optional `:http-port` suffix lets the gateway aggregate status when peers listen on different HTTP ports.

{{< callout type="warn" >}}
All nodes must agree on `SCALAXY_PEERS`; inconsistent topologies produce inconsistent ownership.
{{< /callout >}}

## Health checks

`GET /healthz` returns `{"status":"ok"}` with HTTP 200 when the node's HTTP server is accepting connections. It backs the container `HEALTHCHECK` and the Kubernetes probes.
