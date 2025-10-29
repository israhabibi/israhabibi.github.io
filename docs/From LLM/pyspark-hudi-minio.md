# PySpark + Hudi + MinIO Local Development Setup
## EMR 7.3 Compatible Environment

---

## 🎯 Project Goal

Set up a local development environment that mimics AWS EMR 7.3 for testing PySpark with Hudi, using Docker to avoid Java version conflicts and dependency management issues. The setup includes MinIO for S3-compatible storage and volume mounting for local code development.

---

## 📋 What We Built

### Components
- **PySpark 3.5.0** (EMR 7.3 version)
- **Hudi 0.14.1** (Table format with ACID transactions)
- **MinIO** (S3-compatible object storage)
- **Jupyter Lab** (Interactive development)
- **Delta Lake 3.1.0** (Bonus)
- **Iceberg 1.4.3** (Bonus)

### Architecture
```
┌─────────────────────────────────────────┐
│  Your Local Machine                     │
│  ├── workspace/  (synced) ──────────┐   │
│  ├── notebooks/  (synced) ──────────┤   │
│  ├── scripts/    (synced) ──────────┤   │
│  └── config/     (synced) ──────────┤   │
└─────────────────────────────────────┼───┘
                                      │
                    ┌─────────────────▼──────────────────┐
                    │  Docker Containers                 │
                    │  ┌──────────────────────────────┐  │
                    │  │  PySpark + Hudi              │  │
                    │  │  - Jupyter Lab :8888         │  │
                    │  │  - Spark UI :4040            │  │
                    │  │  - Java 17 + Python 3.11     │  │
                    │  └──────────────────────────────┘  │
                    │  ┌──────────────────────────────┐  │
                    │  │  MinIO (S3)                  │  │
                    │  │  - API :9000                 │  │
                    │  │  - Console :9001             │  │
                    │  │  - Buckets: hudi-bucket,     │  │
                    │  │    delta-bucket, data-lake   │  │
                    │  └──────────────────────────────┘  │
                    └────────────────────────────────────┘
```

---

## 🚀 Complete Setup Instructions

### Step 1: Create Project Structure

```bash
# Create main directory
mkdir pyspark-hudi-emr73
cd pyspark-hudi-emr73

# Create subdirectories for code
mkdir workspace notebooks scripts config
```

---

### Step 2: Create Dockerfile

Create `Dockerfile` with this content:

```dockerfile
# Dockerfile - EMR 7.3 Compatible Setup (Ubuntu Base)
FROM ubuntu:22.04

# Prevent interactive prompts during package installation
ENV DEBIAN_FRONTEND=noninteractive

# Install dependencies and add Python 3.11 repository
RUN apt-get update && apt-get install -y \
    software-properties-common \
    wget \
    curl \
    procps \
    && add-apt-repository ppa:deadsnakes/ppa -y \
    && apt-get update \
    && apt-get install -y \
    openjdk-17-jdk \
    python3.11 \
    python3.11-distutils \
    python3.11-dev \
    && rm -rf /var/lib/apt/lists/*

# Install pip for Python 3.11
RUN curl -sS https://bootstrap.pypa.io/get-pip.py | python3.11

# Set Java and Python as default
ENV JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64
ENV PATH=$JAVA_HOME/bin:$PATH

# Create symbolic links for python
RUN ln -sf /usr/bin/python3.11 /usr/bin/python3 && \
    ln -sf /usr/bin/python3.11 /usr/bin/python

# EMR 7.3 versions
ENV SPARK_VERSION=3.5.0
ENV HADOOP_VERSION=3.3.6
ENV HUDI_VERSION=0.14.1
ENV DELTA_VERSION=3.1.0
ENV ICEBERG_VERSION=1.4.3
ENV SPARK_HOME=/usr/lib/spark
ENV HADOOP_HOME=/usr/lib/hadoop
ENV PATH=$SPARK_HOME/bin:$HADOOP_HOME/bin:$PATH
ENV PYTHONPATH=$SPARK_HOME/python:$SPARK_HOME/python/lib/py4j-0.10.9.7-src.zip:$PYTHONPATH

# Download and install Hadoop
RUN wget -q https://archive.apache.org/dist/hadoop/common/hadoop-${HADOOP_VERSION}/hadoop-${HADOOP_VERSION}.tar.gz \
    && tar -xzf hadoop-${HADOOP_VERSION}.tar.gz -C /usr/lib \
    && mv /usr/lib/hadoop-${HADOOP_VERSION} $HADOOP_HOME \
    && rm hadoop-${HADOOP_VERSION}.tar.gz

# Download and install Spark
RUN wget -q https://archive.apache.org/dist/spark/spark-${SPARK_VERSION}/spark-${SPARK_VERSION}-bin-hadoop3.tgz \
    && tar -xzf spark-${SPARK_VERSION}-bin-hadoop3.tgz -C /usr/lib \
    && mv /usr/lib/spark-${SPARK_VERSION}-bin-hadoop3 $SPARK_HOME \
    && rm spark-${SPARK_VERSION}-bin-hadoop3.tgz

# Install PySpark and other Python dependencies (EMR 7.3 compatible versions)
RUN python3.11 -m pip install --no-cache-dir \
    pyspark==3.5.0 \
    jupyter \
    jupyterlab \
    pandas==2.1.4 \
    numpy==1.26.3 \
    boto3==1.34.34 \
    pyarrow==14.0.1

# Download Hudi bundle (EMR 7.3 uses Spark 3.5)
RUN wget -q https://repo1.maven.org/maven2/org/apache/hudi/hudi-spark3.5-bundle_2.12/${HUDI_VERSION}/hudi-spark3.5-bundle_2.12-${HUDI_VERSION}.jar \
    -P $SPARK_HOME/jars/

# Download Delta Lake jars
RUN wget -q https://repo1.maven.org/maven2/io/delta/delta-spark_2.12/${DELTA_VERSION}/delta-spark_2.12-${DELTA_VERSION}.jar \
    -P $SPARK_HOME/jars/ \
    && wget -q https://repo1.maven.org/maven2/io/delta/delta-storage/${DELTA_VERSION}/delta-storage-${DELTA_VERSION}.jar \
    -P $SPARK_HOME/jars/

# Download Iceberg jars
RUN wget -q https://repo1.maven.org/maven2/org/apache/iceberg/iceberg-spark-runtime-3.5_2.12/${ICEBERG_VERSION}/iceberg-spark-runtime-3.5_2.12-${ICEBERG_VERSION}.jar \
    -P $SPARK_HOME/jars/

# Download AWS SDK jars for S3 support
RUN wget -q https://repo1.maven.org/maven2/org/apache/hadoop/hadoop-aws/${HADOOP_VERSION}/hadoop-aws-${HADOOP_VERSION}.jar \
    -P $SPARK_HOME/jars/ \
    && wget -q https://repo1.maven.org/maven2/com/amazonaws/aws-java-sdk-bundle/1.12.262/aws-java-sdk-bundle-1.12.262.jar \
    -P $SPARK_HOME/jars/

WORKDIR /workspace

# Expose ports
EXPOSE 8888 4040 18080

CMD ["bash"]
```

---

### Step 3: Create docker-compose.yml

Create `docker-compose.yml` with this content:

```yaml
version: '3.8'

services:
  # MinIO - S3 Compatible Storage
  minio:
    image: minio/minio:latest
    container_name: minio
    ports:
      - "9000:9000"      # API port
      - "9001:9001"      # Console port
    environment:
      MINIO_ROOT_USER: ROOTNAME
      MINIO_ROOT_PASSWORD: CHANGEME123
    volumes:
      - minio-data:/data
    command: server /data --console-address ":9001"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 20s
      retries: 3

  # MinIO Client - Create bucket on startup
  minio-setup:
    image: minio/mc:latest
    container_name: minio-setup
    depends_on:
      - minio
    entrypoint: >
      /bin/sh -c "
      sleep 5;
      /usr/bin/mc alias set myminio http://minio:9000 ROOTNAME CHANGEME123;
      /usr/bin/mc mb myminio/hudi-bucket --ignore-existing;
      /usr/bin/mc mb myminio/delta-bucket --ignore-existing;
      /usr/bin/mc mb myminio/data-lake --ignore-existing;
      echo 'Buckets created successfully';
      exit 0;
      "

  # PySpark with Hudi
  pyspark-hudi:
    build: .
    container_name: pyspark-hudi-dev
    depends_on:
      - minio
    volumes:
      - ./workspace:/workspace          # Your code and data
      - ./notebooks:/notebooks          # Jupyter notebooks
      - ./scripts:/scripts              # Python scripts
      - ./config:/config                # Configuration files
    ports:
      - "8888:8888"  # Jupyter
      - "4040:4040"  # Spark UI
      - "18080:18080" # Spark History Server
    environment:
      - JUPYTER_ENABLE_LAB=yes
      - MINIO_ENDPOINT=http://minio:9000
      - MINIO_ACCESS_KEY=ROOTNAME
      - MINIO_SECRET_KEY=CHANGEME123
    command: jupyter lab --ip=0.0.0.0 --port=8888 --no-browser --allow-root --NotebookApp.token='' --NotebookApp.password=''

volumes:
  minio-data:
    driver: local
```

---

### Step 4: Build and Start

```bash
# Build the Docker image (takes 5-10 minutes first time)
docker-compose build

# Start all services
docker-compose up -d

# Verify everything is running
docker-compose ps
```

---

### Step 5: Create Test Script

Create `scripts/test_hudi_minio.py`:

```python
from pyspark.sql import SparkSession
import os

# MinIO Configuration
minio_endpoint = os.getenv("MINIO_ENDPOINT", "http://minio:9000")
minio_access_key = os.getenv("MINIO_ACCESS_KEY", "ROOTNAME")
minio_secret_key = os.getenv("MINIO_SECRET_KEY", "CHANGEME123")

# Create Spark session with Hudi and MinIO
spark = SparkSession.builder \
    .appName("Hudi-MinIO-EMR-7.3") \
    .config("spark.serializer", "org.apache.spark.serializer.KryoSerializer") \
    .config("spark.sql.catalog.spark_catalog", "org.apache.spark.sql.hudi.catalog.HoodieCatalog") \
    .config("spark.sql.extensions", "org.apache.spark.sql.hudi.HoodieSparkSessionExtension") \
    .config("spark.kryo.registrator", "org.apache.spark.HoodieSparkKryoRegistrar") \
    .config("spark.hadoop.fs.s3a.endpoint", minio_endpoint) \
    .config("spark.hadoop.fs.s3a.access.key", minio_access_key) \
    .config("spark.hadoop.fs.s3a.secret.key", minio_secret_key) \
    .config("spark.hadoop.fs.s3a.path.style.access", "true") \
    .config("spark.hadoop.fs.s3a.impl", "org.apache.hadoop.fs.s3a.S3AFileSystem") \
    .config("spark.hadoop.fs.s3a.connection.ssl.enabled", "false") \
    .getOrCreate()

print("✅ Spark Version:", spark.version)
print("✅ MinIO Endpoint:", minio_endpoint)

# Sample data
data = [
    (1, "Alice", 25, "Engineering", "2024-01-01"),
    (2, "Bob", 30, "Sales", "2024-01-01"),
    (3, "Charlie", 35, "Marketing", "2024-01-01")
]

df = spark.createDataFrame(data, ["id", "name", "age", "department", "date"])
print("\n📊 Original Data:")
df.show()

# Hudi configuration for MinIO
table_name = "employee_table"
s3_path = f"s3a://hudi-bucket/{table_name}"

hudi_options = {
    'hoodie.table.name': table_name,
    'hoodie.datasource.write.recordkey.field': 'id',
    'hoodie.datasource.write.partitionpath.field': 'date',
    'hoodie.datasource.write.table.name': table_name,
    'hoodie.datasource.write.operation': 'upsert',
    'hoodie.datasource.write.precombine.field': 'age',
    'hoodie.datasource.write.hive_style_partitioning': 'true',
    'hoodie.upsert.shuffle.parallelism': 2,
    'hoodie.insert.shuffle.parallelism': 2
}

# Write to Hudi table in MinIO
print(f"\n💾 Writing to MinIO: {s3_path}")
df.write.format("hudi") \
    .options(**hudi_options) \
    .mode("overwrite") \
    .save(s3_path)

print("✅ Data written successfully!")

# Read from Hudi table
print(f"\n📖 Reading from MinIO: {s3_path}")
hudi_df = spark.read.format("hudi").load(s3_path)
hudi_df.select("id", "name", "age", "department", "date").show()

# Update operation
update_data = [
    (2, "Bob Smith", 31, "Sales", "2024-01-01"),
    (4, "Diana", 28, "HR", "2024-01-02")
]

update_df = spark.createDataFrame(update_data, ["id", "name", "age", "department", "date"])

update_df.write.format("hudi") \
    .options(**hudi_options) \
    .mode("append") \
    .save(s3_path)

print("\n✅ Upsert completed!")

# Read updated data
updated_hudi_df = spark.read.format("hudi").load(s3_path)
updated_hudi_df.select("id", "name", "age", "department", "date") \
    .orderBy("id") \
    .show()

print("\n🎉 Test completed successfully!")
spark.stop()
```

---

### Step 6: Run Test

```bash
# Run the test script
docker exec pyspark-hudi-dev python3 /scripts/test_hudi_minio.py
```

---

## 🌐 Access Points

| Service | URL | Credentials |
|---------|-----|-------------|
| **Jupyter Lab** | http://localhost:8888 | No password |
| **MinIO Console** | http://localhost:9001 | ROOTNAME / CHANGEME123 |
| **Spark UI** | http://localhost:4040 | (when job running) |

---

## 📂 Volume Mapping

Your local files are automatically synced:

```
Local Directory          →  Container Path
./workspace/            →  /workspace/
./notebooks/            →  /notebooks/
./scripts/              →  /scripts/
./config/               →  /config/
```

**This means:**
- Edit code on your local machine with any IDE
- Changes are instantly available in the container
- No need to rebuild or restart

---

## 💻 Development Workflow

### Option 1: Jupyter Lab (Recommended)
```bash
# Open browser
http://localhost:8888

# Create notebook and start coding
```

### Option 2: Python Scripts
```bash
# Create script locally
vim scripts/my_job.py

# Run in container
docker exec pyspark-hudi-dev python3 /scripts/my_job.py
```

### Option 3: Interactive Shell
```bash
# Access container
docker exec -it pyspark-hudi-dev bash

# Run PySpark shell
pyspark
```

---

## 🔧 Common Commands

```bash
# View logs
docker-compose logs -f pyspark-hudi

# Stop services
docker-compose down

# Stop and remove all data
docker-compose down -v

# Restart services
docker-compose restart

# Rebuild after Dockerfile changes
docker-compose build --no-cache
docker-compose up -d

# Access container shell
docker exec -it pyspark-hudi-dev bash
```

---

## 📊 EMR 7.3 Version Compatibility

| Component | Version | Match EMR 7.3 |
|-----------|---------|---------------|
| Java | 17 | ✅ |
| Python | 3.11 | ✅ |
| Spark | 3.5.0 | ✅ |
| Hadoop | 3.3.6 | ✅ |
| Hudi | 0.14.1 | ✅ |
| Delta Lake | 3.1.0 | ✅ |
| Iceberg | 1.4.3 | ✅ |

---

## 🎯 Key Design Decisions

### Why Ubuntu Base Image?
- ✅ More straightforward package management
- ✅ Better Python 3.11 support via deadsnakes PPA
- ✅ Familiar for most developers
- ✅ Reliable pip installation

### Why MinIO?
- ✅ S3-compatible API (test locally, deploy to S3)
- ✅ Fast local storage
- ✅ Web console for debugging
- ✅ No AWS credentials needed for development

### Why Volume Mounting?
- ✅ Code persists on your machine
- ✅ Use your favorite IDE
- ✅ No need to rebuild on code changes
- ✅ Easy version control with git

---

## 🐛 Troubleshooting

### Port Already in Use
```bash
# Find what's using the port
lsof -i :8888

# Kill it or change port in docker-compose.yml
# Change "8888:8888" to "8889:8888"
```

### Container Won't Start
```bash
# Check logs
docker-compose logs pyspark-hudi

# Ensure Docker has enough resources
# Docker Desktop → Settings → Resources → RAM: 8GB
```

### Can't Connect to MinIO
- Inside container: use `http://minio:9000`
- From host: use `http://localhost:9000`

### Build Fails
```bash
# Clean everything
docker-compose down -v
docker system prune -af

# Rebuild from scratch
docker-compose build --no-cache
```

---

## 🚀 Next Steps

Now you can:
- ✅ Develop Hudi pipelines locally
- ✅ Test complex transformations
- ✅ Debug before deploying to EMR
- ✅ Share environment with team (just share Dockerfile + docker-compose.yml)
- ✅ Version control your entire dev environment

---

## 📚 Additional Resources

- **Hudi Documentation**: https://hudi.apache.org/docs/overview/
- **MinIO Documentation**: https://min.io/docs/minio/linux/index.html
- **Spark Documentation**: https://spark.apache.org/docs/3.5.0/
- **EMR 7.3 Release Notes**: https://docs.aws.amazon.com/emr/latest/ReleaseGuide/emr-730-release.html

---

## ✨ Summary

You now have a complete local EMR 7.3 environment with:
- ✅ PySpark + Hudi for lakehouse operations
- ✅ MinIO for S3-compatible storage
- ✅ Jupyter Lab for interactive development
- ✅ Volume mounting for seamless code editing
- ✅ No Java/Python version conflicts
- ✅ Production-like testing environment

**Happy Coding! 🎉**