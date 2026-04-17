---
title: "PySpark + Hudi + MinIO Local Development Setup"
date: 2025-09-15
authors:
  - isra
tags:
  - pyspark
  - hudi
  - minio
  - data-engineering
  - docker
  - emr
---

# PySpark + Hudi + MinIO Local Development Setup

An EMR 7.3-compatible local development environment built with Docker, so you can test PySpark jobs against Hudi tables on S3-compatible storage without spinning up a real EMR cluster.

<!-- more -->

## Context

Running Spark jobs on EMR is expensive and slow to iterate on. The goal here: reproduce the EMR 7.3 runtime locally — same Spark, Hadoop, Java, and Python versions — and point it at MinIO instead of S3. This keeps the dev loop tight (edit → run → inspect) while catching version-specific bugs that a generic PySpark image would hide.

---

## Stack

| Component | Version | Purpose |
|---|---|---|
| PySpark | 3.5.0 | Matches EMR 7.3 |
| Hadoop | 3.3.6 | S3A client + Hadoop libs |
| Hudi | 0.14.1 | ACID table format |
| Java | 17 | EMR-aligned JVM |
| Python | 3.11 | EMR-aligned interpreter |
| MinIO | latest | S3-compatible object store |
| Jupyter Lab | latest | Interactive notebooks |

Optional: Delta Lake 3.1.0 and Iceberg 1.4.3 bundles can be swapped in.

---

## Project Structure

```
pyspark-hudi-local/
├── Dockerfile
├── docker-compose.yml
├── workspace/         # mounted into container
├── notebooks/         # Jupyter notebooks
├── scripts/           # standalone PySpark scripts
└── config/            # spark-defaults, core-site.xml
```

---

## Dockerfile (Outline)

The base image is **Ubuntu 22.04**. On top of it:

1. Install Java 17 + Python 3.11 from apt.
2. Download Spark 3.5.0 + Hadoop 3.3.6 binaries.
3. Pull Hudi, AWS SDK, and Hadoop-AWS JARs from Maven Central into `$SPARK_HOME/jars`.
4. Install `pyspark`, `jupyterlab`, and any analysis libs with pip.

Ubuntu is chosen over `openjdk` slim images because the apt ecosystem makes it easier to install exact Python + Java pairings without building from source.

---

## docker-compose.yml

Two services — **minio** and **pyspark** — share a Docker network.

```yaml
services:
  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    ports:
      - "9000:9000"
      - "9001:9001"
    environment:
      MINIO_ROOT_USER: ROOTNAME
      MINIO_ROOT_PASSWORD: CHANGEME123
    volumes:
      - minio-data:/data

  minio-init:
    image: minio/mc
    depends_on:
      - minio
    entrypoint: >
      /bin/sh -c "
      mc alias set local http://minio:9000 ROOTNAME CHANGEME123;
      mc mb -p local/warehouse;
      exit 0;
      "

  pyspark:
    build: .
    ports:
      - "8888:8888"   # Jupyter
      - "4040:4040"   # Spark UI
    volumes:
      - ./workspace:/workspace
      - ./notebooks:/notebooks
      - ./scripts:/scripts
    environment:
      AWS_ACCESS_KEY_ID: ROOTNAME
      AWS_SECRET_ACCESS_KEY: CHANGEME123
    depends_on:
      - minio-init

volumes:
  minio-data:
```

The `minio-init` container auto-creates a `warehouse` bucket on first boot — no manual MinIO console clicking required.

---

## Smoke Test: Write + Read a Hudi Table

```python
from pyspark.sql import SparkSession

spark = (
    SparkSession.builder
    .appName("hudi-smoke")
    .config("spark.serializer", "org.apache.spark.serializer.KryoSerializer")
    .config("spark.sql.extensions", "org.apache.spark.sql.hudi.HoodieSparkSessionExtension")
    .config("spark.sql.catalog.spark_catalog",
            "org.apache.spark.sql.hudi.catalog.HoodieCatalog")
    .config("spark.hadoop.fs.s3a.endpoint", "http://minio:9000")
    .config("spark.hadoop.fs.s3a.path.style.access", "true")
    .config("spark.hadoop.fs.s3a.impl", "org.apache.hadoop.fs.s3a.S3AFileSystem")
    .getOrCreate()
)

df = spark.createDataFrame(
    [(1, "alice", 100), (2, "bob", 200)],
    ["id", "name", "amount"],
)

(df.write.format("hudi")
    .option("hoodie.table.name", "users")
    .option("hoodie.datasource.write.recordkey.field", "id")
    .option("hoodie.datasource.write.precombine.field", "id")
    .mode("overwrite")
    .save("s3a://warehouse/users"))

spark.read.format("hudi").load("s3a://warehouse/users").show()
```

Upsert works the same way — point at the same path with `mode("append")` and Hudi handles merge semantics.

---

## Access Points

| Service | URL | Credentials |
|---|---|---|
| Jupyter Lab | `http://localhost:8888` | token printed in container logs |
| MinIO Console | `http://localhost:9001` | ROOTNAME / CHANGEME123 |
| Spark UI | `http://localhost:4040` | — |

---

## Why This Setup

- **Ubuntu base** — predictable apt-based install of matched Java + Python versions; avoids glibc issues seen on Alpine.
- **MinIO over fake S3 libs** — real S3 protocol, catches `s3a://` edge cases before they bite in EMR.
- **Volume mounts for `workspace/` and `scripts/`** — edit with any IDE on the host, no container rebuild needed for code changes.
- **Maven-pulled JARs at build time** — JAR versions locked in the image, not lazily downloaded at runtime.

---

## Troubleshooting

!!! bug "Port already in use"
    `8888` (Jupyter) and `9000` (MinIO) are common conflicts. Change the host-side port in `docker-compose.yml`.

!!! bug "MinIO bucket not created"
    If the `minio-init` container exited before MinIO was ready, restart it: `docker compose up -d minio-init`.

!!! bug "`ClassNotFoundException` on Hudi ops"
    The Hudi bundle JAR is missing or version-mismatched. Verify it's in `$SPARK_HOME/jars` inside the container.

---

## Key Takeaways

- Matching EMR versions locally catches runtime bugs 2–3 days earlier than deploying to a dev cluster.
- MinIO + S3A is a faithful S3 stand-in for Hudi/Delta/Iceberg — use it instead of fake filesystem layers.
- Keeping JARs pre-baked in the image is faster and more reproducible than downloading via `--packages` at runtime.
- Volume mounts over `COPY` — rebuild only when dependencies change, not when code changes.
