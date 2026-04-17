---
title: "ClickHouse: Docker vs Kubernetes (k3d) Benchmark"
date: 2026-04-16
authors:
  - isra
tags:
  - clickhouse
  - kubernetes
  - docker
  - benchmark
  - data-engineering
---

# ClickHouse: Docker vs Kubernetes (k3d) — Benchmark

Head-to-head performance comparison of ClickHouse running in plain **Docker**
vs ClickHouse on **k3d** (Kubernetes-in-Docker), using a 10M-row synthetic
OLAP dataset on the same MacBook.

<!-- more -->

## Context

This is a local proof-of-concept benchmark to evaluate whether running ClickHouse
inside a Kubernetes cluster (via k3d) introduces meaningful query overhead
compared to a plain Docker container — relevant when deciding deployment
strategy for a self-hosted OLAP layer.

Both instances ran on the **same machine** (MacBook, Apple Silicon), same
ClickHouse version (`26.3.9.8`), same dataset.

---

## Setup

### Instances

| | Docker | k3d (Kubernetes) |
|---|---|---|
| Version | `26.3.9.8` | `26.3.9.8` |
| HTTP Port | `localhost:8123` | `localhost:18123` |
| Resources | unconstrained | 1 CPU / 2Gi limit |
| k8s cluster | — | k3d `clickhouse-bench`, 1 agent + 1 server |

### k3d Fix Journey

The k3d cluster was already running but had **no pods** — only a namespace and
Service existed. Three issues were found and fixed:

!!! bug "Issue 1 — No StatefulSet"
    The pod was never created. Fixed by applying a StatefulSet manifest with
    `app=clickhouse` labels matching the existing Service selector.

!!! bug "Issue 2 — Auth disabled"
    Empty `CLICKHOUSE_PASSWORD` env var caused ClickHouse to disable network
    access entirely. Fixed by patching the StatefulSet env to set a real password.

!!! bug "Issue 3 — nginx routing wrong port"
    The k3d loadbalancer nginx was routing `host:18123 → node:8123` (direct port,
    nothing listening) instead of `node:30123` (NodePort). Manually patched
    `/etc/nginx/nginx.conf` inside the LB container and reloaded nginx.

### Dataset

```
bm_transactions   10,000,000 rows  (id, user_id, product_id, amount, status, ts)
bm_users             100,000 rows  (user_id, name, age, gender, city, registered)
bm_products            1,000 rows  (product_id, name, category, price)
```

CSV size: ~516 MB for transactions. Generated with `random.Random(seed=42)`.

---

## Ingest Performance

| Metric | Docker | k3d |
|---|---|---|
| 10M rows loaded | **46,498 ms** | 120,379 ms |
| Ingest speed | **~215K rows/s** | ~83K rows/s |
| Ratio | **2.6× faster** | — |

---

## Query Benchmark Results

All queries: **best of 3 warm runs**.

| Query | Docker | k3d | Winner | Delta |
|---|---|---|---|---|
| Q1 — COUNT + AVG by status | **661 ms** | 2,824 ms | Docker | 4.3× |
| Q2 — Date range + SUM (1 quarter) | **311 ms** | 3,721 ms | Docker | 12.0× |
| Q3 — JOIN txn × products | **610 ms** | 3,791 ms | Docker | 6.2× |
| Q4 — 3-table JOIN + GROUP BY | **5,854 ms** | 12,039 ms | Docker | 2.1× |
| Q5 — Window: top-10 users by spend | **322 ms** | 10,450 ms | Docker | 32.5× |
| Q6 — Failed txn ratio per city | 3,705 ms | **3,132 ms** | k3d | 1.2× |
| Q7 — Monthly revenue trend | **81 ms** | 373 ms | Docker | 4.6× |
| Q8 — Full scan COUNT | 10 ms | **9 ms** | k3d | ~1.0× |

### Summary

| | Docker | k3d |
|---|---|---|
| Total query time | **11,554 ms** | 36,340 ms |
| Query wins | **6 / 8** | 2 / 8 |
| Overall | **3.1× faster** | — |

---

## Analysis

Docker wins 6/8 queries, with the largest gaps on:

- **Q5 (Window function): 32.5×** — parallelism throttled by k3d CPU limit
- **Q2 (Date range scan): 12.0×** — I/O and CPU both constrained in k3d

The overhead in k3d comes from three layers:

1. **Network path** — `localhost → LB container → NodePort → pod` vs direct container
2. **CPU limits** — StatefulSet capped at 1 core, crippling analytical parallelism
3. **cgroup overhead** — Kubernetes adds per-container CPU/memory accounting

!!! tip "Key takeaway for production k8s"
    If running ClickHouse on Kubernetes for OLAP, **remove or raise the CPU limit**.
    Capping at 1 core is the single biggest performance killer for analytical queries.

---

## Deployment Decision Guide

| Scenario | Recommendation |
|---|---|
| Local dev / POC | Plain Docker — simpler, faster |
| Multi-node / HA | Kubernetes — needed for replication |
| Resource isolation | Kubernetes with tuned limits |
| Max query performance on k8s | No CPU cap, use `Guaranteed` QoS class |

---

## Environment

- **Host**: MacBook (Apple Silicon, macOS)
- **Docker**: Docker Desktop + k3d `v5.8.3`
- **k3s**: `v1.31.5+k3s1`
- **ClickHouse**: `26.3.9.8`
- **Benchmark date**: 2026-04-16
