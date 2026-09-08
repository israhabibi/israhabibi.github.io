---
title: About
---

# Meet Isra

**Isra Nurul Habibi** — Data Engineer.

I build data pipelines, turn raw data into something reliable, and write about what I learn along the way.

---

## What I Do

I work as a Data Engineer — designing and maintaining data infrastructure that powers analytics, products, and decisions.

My day-to-day revolves around:

- **Data pipelines** — building and operating batch/streaming pipelines that move data from source to destination reliably.
- **Large-scale processing** — PySpark, SQL, and distributed compute for workloads that don't fit in a single machine.
- **Infrastructure** — Airflow for orchestration, EMR, ClickHouse for analytics storage, Redshift for warehouse workloads.
- **Observability & quality** — making sure data actually arrives, is correct, and can be trusted.

I care about systems that are maintainable, observable, and not unnecessarily complex.

---

## What I Write About

This blog is a record of things I figure out — the useful parts, the failures, and the investigations in between.

Topics I cover:

- **Data Engineering** — pipeline patterns, Spark pitfalls, SQL performance, warehouse vs analytics DB choices.
- **Infra & Homelab** — Kubernetes (k3s/k3d), self-hosted setups, HA experiments, cost-aware infra.
- **Tooling** — the tools and workflows I reach for daily, and why.

Posts range from full write-ups (benchmarks, deep dives, step-by-step setup guides) to shorter TIL notes for things too small for a full post but still worth keeping.

---

## Selected Work

Some of what I've written and built:

### Blog

- **ClickHouse: Docker vs Kubernetes (k3d) — Benchmark** — head-to-head OLAP performance on the same dataset, Docker vs k3d.
- **PySpark + Hudi + MinIO Local Development Setup** — EMR-compatible local dev environment with Docker, no real EMR cluster needed.
- **K3s on GCP: Free-Tier Single Node to 3-Node Cluster** — going from one node to a 3-node cluster on GCP free tier.
- **2-Laptop HA Homelab — Auto-Failover with Cloudflare + Syncthing** — high-availability setup across two old laptops.
- **Wake-on-LAN on an Ubuntu Laptop Server** — waking a laptop-server remotely.
- **Setting Up an Old Ubuntu Desktop as a Home Server** — turning retired hardware into something useful.
- **Supervisor Manager — One-Command Python App Deployment** — simple deployment tooling for Python services.
- **How I Use Claude as a Data Engineer** — how I use AI in my daily data engineering workflow.

### TIL Notes

Short notes worth remembering:

- ClickHouse empty password disables network access.
- k3d nginx routes to container port, not NodePort.
- MySQL: wrapping a column in a function breaks index usage.
- MySQL CASCADE + SET NULL on the same row — second constraint is moot.
- Airflow DAG version churn — sort your config dict.

---

## Stack I Use

Things I reach for regularly:

| Area | Tools |
|------|-------|
| Processing | Python, PySpark, SQL |
| Orchestration | Airflow 3, EMR on EKS |
| Storage | S3, Redshift, ClickHouse, MySQL 8, Hudi |
| Infra | k3d, Docker, Kubernetes |
| Observability | Dynatrace, Metabase |

---

## Contact

If you want to get in touch:

- **GitHub**: [@israhabibi](https://github.com/israhabibi)
- **Email**: [israhabibi@gmail.com](mailto:israhabibi@gmail.com)

Built with [MkDocs](https://www.mkdocs.org/) and [Material for MkDocs](https://squidfunk.github.io/mkdocs-material/).
