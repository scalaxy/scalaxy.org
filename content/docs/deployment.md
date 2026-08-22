---
date: 2026-08-06
lastmod: 2026-08-21
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
| `SCALAXY_WEB_DIR` | `web/` | Console static files |
| `SCALAXY_STORE_BACKEND` | *(none)* | Set to `s3` for S3 object storage |
| `SCALAXY_S3_ENDPOINT` | *(none)* | S3-compatible endpoint URL |
| `SCALAXY_S3_BUCKET` | *(none)* | Per-node S3 bucket |
| `SCALAXY_S3_ACCESS_KEY` | *(none)* | S3 access key |
| `SCALAXY_S3_SECRET_KEY` | *(none)* | S3 secret key |
| `SCALAXY_S3_REGION` | `us-east-1` | S3 region |
| `SCALAXY_S3_PREFIX` | `scalaxy/` | Key prefix (node ID appended automatically) |
| `SCALAXY_S3_LAZY` | `false` | Enable lazy loading and aggregate summaries |
| `SCALAXY_S3_ENCRYPTION_KEY` | *(none)* | Encrypt all S3 objects at rest |
| `SCALAXY_S3_CACHE_MAX_BYTES` | unlimited | Local cache eviction budget |

## Docker compose example

Three-node cluster with Garage S3, lazy loading, and encryption:

```yaml
services:
  garage:
    image: dxflrs/garage:v1.0.0
    # ... Garage configuration ...

  scalaxy-0:
    image: scalaxy:integration
    environment:
      SCALAXY_NODE_ID: node-0
      SCALAXY_STORE_BACKEND: s3
      SCALAXY_S3_ENDPOINT: http://garage:3900
      SCALAXY_S3_BUCKET: int-0
      SCALAXY_S3_ACCESS_KEY: GK...
      SCALAXY_S3_SECRET_KEY: ...
      SCALAXY_S3_PREFIX: myapp
      SCALAXY_S3_LAZY: "true"
      SCALAXY_S3_ENCRYPTION_KEY: "your-secret-key-here"
      SCALAXY_PEERS: node-0=scalaxy-0:7200,node-1=scalaxy-1:7200,node-2=scalaxy-2:7200
      SCALAXY_REPLICATE_TO: node-1=scalaxy-1:7200

  # ... scalaxy-1 and scalaxy-2 with their own buckets and keys ...
```

## Encryption

Setting `SCALAXY_S3_ENCRYPTION_KEY` enables authenticated encryption of
all data stored in S3. Each object body is encrypted with HMAC-SHA256
CTR-mode encryption before upload and decrypted transparently on read.

The key can be any non-empty string; it is converted to octets internally.
All nodes must use the same key to read each other's data.

Without the key, data is stored unencrypted (backward compatible).

## Local persistent cache

Each node maintains a local disk cache of fetched S3 objects under
`$SCALAXY_DATA_DIR/s3-cache/`. The cache stores plaintext (decrypted)
data for fast reads; it is on the trusted node's local disk.

Cache eviction uses a budget set by `SCALAXY_S3_CACHE_MAX_BYTES`.
When unset, the cache grows without bound (recommended for dedicated
nodes with sufficient disk).
