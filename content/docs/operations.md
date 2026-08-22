---
date: 2026-08-06
lastmod: 2026-08-17
title: "Operations"
description: "Health checks, failure behavior, durability, and troubleshooting."
weight: 90
---

## Monitoring

- **Health**: poll `/healthz`; non-200 means the node is not serving.
- **Status**: `/api/status` returns cluster state: `healthy`, `degraded`, or node-level `unreachable`.
- **Metrics**: per-node key counts and uptime via `/api/node-status` (used by the dashboard's cluster view).

## Failure behavior

| Event | What happens |
|---|---|
| Node crash | Log replayed on restart; acknowledged writes survive. |
| Node unreachable | Reads fail over to replicas; writes go to a reachable member; status shows `degraded`. |
| Node rejoin | Ownership resumes; status returns to `healthy`. |
| Disk full | Writes to the append-only log fail; the node reports errors on the data plane. |

## Durability

- Every acknowledged write is appended to `scalaxy.log` in `SCALAXY_DATA_DIR` before the client receives the ack.
- The log is the same record format as the network protocol: replay is exact.
- Graph entities (nodes, relationships, properties) are ordinary records in
  the same store, so `CREATE`/`MERGE`/`SET`/`DELETE` mutations ride the
  same durability and replication path as keys.
- Mount `SCALAXY_DATA_DIR` on persistent storage (a Docker volume, a Kubernetes PVC) to survive container restarts.

## Troubleshooting

### "peer unreachable" in the console

- Verify the peer's data port and HTTP port are reachable (`curl http://<peer>:8080/healthz`).
- Check that `SCALAXY_PEERS` matches on every node, including the optional `:http-port` suffix when ports differ.
- Hostnames are resolved at connection time, so DNS names work as-is in containers and Kubernetes.

### Keys missing after a restart

- Confirm `SCALAXY_DATA_DIR` points at persistent storage and that the directory contains `scalaxy.log`.
- The log filename does not depend on the node id: the store is always `scalaxy.log` in the data directory.

### Writes fail with empty cluster

- The gateway requires at least one reachable peer. If all peers report `unreachable`, fix connectivity before writing.

### The test suite fails in CI

- The suite uses real sockets on ephemeral ports and real threads. Run `make test` locally; if it passes and CI fails, check the CI image's SBCL version and network sandbox.
