---
date: 2026-08-06
lastmod: 2026-08-06
title: "Kubernetes"
description: "Run a replicated Scalaxy cluster as a StatefulSet with PVCs and probes."
weight: 80
---

## Deploy

```sh
kubectl apply -f deploy/kubernetes/scalaxy.yaml
kubectl -n scalaxy get pods
```

The manifest creates:

- **Namespace** `scalaxy`
- **ConfigMap** — `SCALAXY_ADDRESS`, `SCALAXY_HTTP_ADDRESS`, `SCALAXY_DATA_DIR`, `SCALAXY_PEERS` (three headless-service DNS names), `SCALAXY_REPLICATE_TO`
- **Headless Service** `scalaxy-headless` — peer discovery: `scalaxy-0.scalaxy-headless`, `scalaxy-1.scalaxy-headless`, …
- **ClusterIP Service** `scalaxy` — client access (port 80 → 8080, plus 7200)
- **StatefulSet** `scalaxy` — 3 replicas, stable identity, one PVC per pod
- **Probes** — readiness and liveness on `GET /healthz`
- **Security** — non-root uid/gid 1000, fsGroup 1000

## Access the console

```sh
kubectl -n scalaxy port-forward svc/scalaxy 8080:80
# http://localhost:8080
```

An optional Ingress is provided in `deploy/kubernetes/ingress.yaml`.

## Node identity

Each pod takes its node id from its pod name (`SCALAXY_NODE_ID` ← `metadata.name`), so `scalaxy-0` is `node-0` on the ring. The ConfigMap's peer list must match the StatefulSet replica count.

## Scaling

1. Edit the ConfigMap so `SCALAXY_PEERS` includes the new pods.
2. Scale the StatefulSet:

```sh
kubectl -n scalaxy scale statefulset scalaxy --replicas=5
```

New pods start, join the ring, and take ownership of their share of the keyspace. Because writes replicate synchronously, existing data is not lost when a pod is drained — but scale **down** carefully: stop traffic first so owners can be drained, or accept that removed pods' keys are re-homed (the in-process cluster API re-homes automatically; the standalone gateway routes to whatever members remain).

## Failure behavior

- A pod restart replays its durability log from its PVC — acknowledged writes survive.
- An unreachable pod degrades the cluster status but reads fail over to replica holders.
- The console on any pod shows the full aggregated cluster.

See [Operations](/docs/operations/) for monitoring and troubleshooting.
