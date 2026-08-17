---
title: "Topologies"
description: "Deployment shapes for Scalaxy: single node, replicated cluster, and Kubernetes."
kicker: "Topologies"
weight: 30
date: 2026-08-06
lastmod: 2026-08-17
---

Scalaxy runs the same code in every topology; only the environment variables change. All topologies use the data-plane port `7200` (binary protocol) and the console port `8080` (web UI, REST API, `/healthz`). Graph entities and Cypher queries travel over the same ring and the same ports as keys — a cluster is a graph database, not a separate deployment.

## Single node

The simplest setup: one node with a durable log and its own web console. Good for development, edge devices, and small workloads.

<div class="diagram">
<svg viewBox="0 0 640 200" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Single node topology">
  <defs>
    <marker id="arr-l" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#2563eb"/>
    </marker>
  </defs>
  <rect x="30" y="50" width="150" height="100" rx="10" fill="#ffffff" stroke="#cbd5e1"/>
  <text x="105" y="92" text-anchor="middle" fill="#0f172a" font-size="14" font-weight="600">Client</text>
  <text x="105" y="112" text-anchor="middle" fill="#94a3b8" font-size="11">REST / TCP</text>
  <line x1="180" y1="100" x2="255" y2="100" stroke="#2563eb" stroke-width="1.8" marker-end="url(#arr-l)"/>
  <rect x="260" y="50" width="180" height="100" rx="10" fill="#eff6ff" stroke="#93c5fd"/>
  <text x="350" y="90" text-anchor="middle" fill="#0f172a" font-size="14" font-weight="600">Scalaxy node</text>
  <text x="350" y="110" text-anchor="middle" fill="#64748b" font-size="11">7200 data · 8080 console</text>
  <text x="350" y="128" text-anchor="middle" fill="#15803d" font-size="11">durable log + web UI</text>
  <line x1="440" y1="100" x2="500" y2="100" stroke="#15803d" stroke-width="1.8" marker-end="url(#arr-l)"/>
  <rect x="505" y="50" width="110" height="100" rx="10" fill="#f8fafc" stroke="#e2e8f0"/>
  <text x="560" y="95" text-anchor="middle" fill="#64748b" font-size="11">data dir</text>
  <text x="560" y="112" text-anchor="middle" fill="#94a3b8" font-size="10" font-family="monospace">scalaxy.log</text>
</svg>
</div>

```sh
bin/scalaxy --address 127.0.0.1:7200 --http-address 127.0.0.1:8080
# or with Docker:
docker run -p 8080:8080 -p 7200:7200 -v scalaxy-data:/var/lib/scalaxy scalaxy
```

## Cluster

A ring of N nodes. Each key is owned by exactly one node (consistent hashing with virtual nodes); writes replicate synchronously to the next node in ring order, and reads fail over to replica holders when an owner is unreachable.

<div class="diagram">
<svg viewBox="0 0 640 320" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Cluster topology with ring">
  <defs>
    <marker id="arr-c" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#2563eb"/>
    </marker>
  </defs>
  <circle cx="320" cy="160" r="110" fill="none" stroke="#e2e8f0" stroke-width="2" stroke-dasharray="6 6"/>
  <g>
    <circle cx="320" cy="160" r="110" fill="none" stroke="#2563eb" stroke-width="14" stroke-dasharray="230 460" stroke-dashoffset="-20" opacity="0.3"/>
    <circle cx="320" cy="160" r="110" fill="none" stroke="#15803d" stroke-width="14" stroke-dasharray="230 460" stroke-dashoffset="-250" opacity="0.3"/>
    <circle cx="320" cy="160" r="110" fill="none" stroke="#94a3b8" stroke-width="14" stroke-dasharray="230 460" stroke-dashoffset="-480" opacity="0.35"/>
  </g>
  <rect x="268" y="22" width="104" height="54" rx="9" fill="#eff6ff" stroke="#93c5fd"/>
  <text x="320" y="46" text-anchor="middle" fill="#0f172a" font-size="12" font-weight="600">node-0</text>
  <text x="320" y="62" text-anchor="middle" fill="#64748b" font-size="10">owner + console</text>
  <rect x="80" y="150" width="104" height="54" rx="9" fill="#f0fdf4" stroke="#86efac"/>
  <text x="132" y="174" text-anchor="middle" fill="#0f172a" font-size="12" font-weight="600">node-1</text>
  <text x="132" y="190" text-anchor="middle" fill="#64748b" font-size="10">replica</text>
  <rect x="456" y="150" width="104" height="54" rx="9" fill="#f8fafc" stroke="#cbd5e1"/>
  <text x="508" y="174" text-anchor="middle" fill="#0f172a" font-size="12" font-weight="600">node-2</text>
  <text x="508" y="190" text-anchor="middle" fill="#64748b" font-size="10">replica</text>
  <path d="M 310 78 Q 200 118 152 148" fill="none" stroke="#2563eb" stroke-width="1.5" marker-end="url(#arr-c)" opacity="0.75"/>
  <path d="M 196 182 Q 285 228 360 212" fill="none" stroke="#15803d" stroke-width="1.5" marker-end="url(#arr-c)" opacity="0.75"/>
  <path d="M 498 204 Q 420 92 344 80" fill="none" stroke="#64748b" stroke-width="1.5" marker-end="url(#arr-c)" opacity="0.75"/>
  <rect x="30" y="265" width="580" height="40" rx="9" fill="#f8fafc" stroke="#e2e8f0"/>
  <text x="320" y="289" text-anchor="middle" fill="#64748b" font-size="11">every node routes to ring owners and serves the full cluster console</text>
</svg>
</div>

```sh
export SCALAXY_PEERS="node-0=scalaxy-0:7200:8080,node-1=scalaxy-1:7200:8080,node-2=scalaxy-2:7200:8080"
export SCALAXY_REPLICATE_TO="node-1=scalaxy-1:7200"
```

{{< callout type="success" title="Verified behavior" >}}
With 3 nodes and one replica per key, stopping any node loses **zero** keys — reads fail over to replicas, writes continue on the survivors, and the cluster reports `degraded` until the node rejoins.
{{< /callout >}}

## Kubernetes

A `StatefulSet` of three replicas with stable pod identity (`scalaxy-0` … `scalaxy-2`), one `PersistentVolumeClaim` per node, and peer discovery through a headless service. Readiness and liveness probes hit `/healthz`; the whole thing runs as a non-root user.

<div class="diagram">
<svg viewBox="0 0 640 230" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Kubernetes topology">
  <rect x="20" y="14" width="600" height="200" rx="12" fill="#f8fafc" stroke="#e2e8f0"/>
  <text x="40" y="40" fill="#94a3b8" font-size="11" font-family="monospace">namespace: scalaxy</text>
  <rect x="40" y="56" width="180" height="56" rx="9" fill="#ffffff" stroke="#93c5fd"/>
  <text x="130" y="80" text-anchor="middle" fill="#0f172a" font-size="12" font-weight="600">scalaxy-0 · pod</text>
  <text x="130" y="97" text-anchor="middle" fill="#64748b" font-size="10">7200 + 8080 · PVC</text>
  <rect x="240" y="56" width="180" height="56" rx="9" fill="#ffffff" stroke="#86efac"/>
  <text x="330" y="80" text-anchor="middle" fill="#0f172a" font-size="12" font-weight="600">scalaxy-1 · pod</text>
  <text x="330" y="97" text-anchor="middle" fill="#64748b" font-size="10">7200 + 8080 · PVC</text>
  <rect x="440" y="56" width="160" height="56" rx="9" fill="#ffffff" stroke="#cbd5e1"/>
  <text x="520" y="80" text-anchor="middle" fill="#0f172a" font-size="12" font-weight="600">scalaxy-2 · pod</text>
  <text x="520" y="97" text-anchor="middle" fill="#64748b" font-size="10">7200 + 8080 · PVC</text>
  <rect x="40" y="132" width="560" height="54" rx="9" fill="#f1f5f9" stroke="#cbd5e1"/>
  <text x="320" y="154" text-anchor="middle" fill="#0f172a" font-size="12" font-weight="600">scalaxy-headless · service</text>
  <text x="320" y="172" text-anchor="middle" fill="#64748b" font-size="10">pod discovery via DNS · probes on /healthz · ingress → svc/scalaxy:80</text>
</svg>
</div>

```sh
kubectl apply -f deploy/kubernetes/scalaxy.yaml
kubectl -n scalaxy port-forward svc/scalaxy 8080:80   # web console
```

See the [Kubernetes guide](/docs/kubernetes/) for the full manifest walkthrough.
