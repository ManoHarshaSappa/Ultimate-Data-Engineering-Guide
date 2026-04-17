# Portfolio Projects: Build Real DE Experience

**Author:** Mano Harsha Sappa | [LinkedIn](https://www.linkedin.com/in/manoharshasappa/) | [Portfolio](https://manoharshasappa.github.io/portfolio_ManoHarshaSappa/) | [Email](mailto:sappamanoharsha@gmail.com)

---

## Contents

1. [Why Projects Matter More Than Certifications](#1-why-projects-matter-more-than-certifications)
2. [What Makes a Great Portfolio Project?](#2-what-makes-a-great-portfolio-project)
3. [Beginner Projects (0-6 months)](#3-beginner-projects-0-6-months)
4. [Intermediate Projects (6-18 months)](#4-intermediate-projects-6-18-months)
5. [Advanced Projects (18+ months)](#5-advanced-projects-18-months)
6. [Free Public Datasets](#6-free-public-datasets)
7. [Project Infrastructure Setup](#7-project-infrastructure-setup)
8. [How to Present Projects to Employers](#8-how-to-present-projects-to-employers)

---

## 1. Why Projects Matter More Than Certifications

A certification tells an employer you passed a multiple-choice test. A portfolio project shows you built something real.

**What hiring managers actually want to see:**

> "I don't care about your AWS certification. Show me a GitHub repo where you built a Kafka → Spark → Delta Lake pipeline. That tells me you actually know what you're doing."
> — Senior DE hiring manager, mid-sized tech company

Projects prove:
- **You can start from zero** (environment setup, config, debugging)
- **You can handle real data** (messy formats, nulls, duplicates, schema changes)
- **You can make architecture decisions** (why did you choose Airflow over a cron job?)
- **You can document your work** (README that explains the what, why, and how)

Certifications complement projects — they don't replace them.

---

## 2. What Makes a Great Portfolio Project?

### The Anatomy of a Strong Project

Every project should have:

**1. A clear README with:**
```markdown
## Problem Statement
What real problem does this solve? Why would a business care?

## Architecture
[Diagram or ASCII art showing data flow]
Source → Tool → Storage → Query Layer → Consumers

## Tech Stack
- Ingestion: [tool and why you chose it]
- Processing: [tool and why]
- Storage: [tool and why]
- Orchestration: [tool and why]
- Visualization: [tool and why]

## Key Design Decisions
- Why Kafka instead of a batch load? Because events arrive continuously.
- Why Delta Lake instead of raw Parquet? For ACID writes and time travel.

## How to Run
[Step-by-step docker-compose up instructions that actually work]

## What I Learned
Honest reflection: what was harder than expected, what you'd do differently.
```

**2. Working code** — not "I built this" but actual runnable code in GitHub

**3. A dataset that has real-world characteristics** — messy data, late arrivals, schema changes

**4. Tests** — at least dbt tests or Great Expectations checks

**5. A LinkedIn post** walking through the project — 5-10 bullet points with a screenshot

---

## 3. Beginner Projects (0-6 months)

### Project 1: NYC Taxi Data Pipeline

**Difficulty:** ⭐⭐ | **Time:** 1-2 weeks | **Resume Impact:** Good

**What you build:** An end-to-end batch pipeline that ingests NYC Yellow Taxi trip data, loads it to PostgreSQL, transforms it with dbt, and visualizes it in a simple dashboard.

**Architecture:**
```
NYC Taxi CSV (public S3 bucket)
    ↓ Python script (pandas chunked read)
PostgreSQL (local Docker)
    ↓ dbt (staging → dimensional model)
├── stg_taxi__trips.sql
├── dim_locations.sql
└── fct_daily_trips.sql (daily aggregates)
    ↓
Metabase or pgAdmin (simple dashboards)
```

**Tech stack:** Python + Pandas + PostgreSQL + dbt + Docker + Metabase

**Step-by-step:**
```python
# ingest.py
import pandas as pd
from sqlalchemy import create_engine
import requests

def ingest_taxi_data(year: int, month: int, engine):
    url = f"https://d37ci6vzurychx.cloudfront.net/trip-data/yellow_tripdata_{year}-{month:02d}.parquet"
    df = pd.read_parquet(url)
    
    # Basic cleaning
    df = df.dropna(subset=["tpep_pickup_datetime", "tpep_dropoff_datetime"])
    df = df[df["trip_distance"] > 0]
    df = df[df["fare_amount"] > 0]
    df["pickup_date"] = df["tpep_pickup_datetime"].dt.date
    
    # Load to Postgres in chunks
    df.to_sql(
        "raw_taxi_trips",
        engine,
        if_exists="append",
        index=False,
        chunksize=10000
    )
    print(f"Loaded {len(df):,} trips for {year}-{month:02d}")

engine = create_engine("postgresql://postgres:password@localhost:5432/taxi_db")
for month in range(1, 13):
    ingest_taxi_data(2024, month, engine)
```

**dbt model:**
```sql
-- fct_daily_trips.sql
{{ config(materialized='table') }}

WITH source AS (
    SELECT *
    FROM {{ ref('stg_taxi__trips') }}
    WHERE pickup_date >= '2024-01-01'
)

SELECT
    pickup_date,
    COUNT(*) AS total_trips,
    SUM(fare_amount) AS total_revenue,
    AVG(trip_distance) AS avg_distance_miles,
    AVG(fare_amount) AS avg_fare,
    SUM(tip_amount) / NULLIF(SUM(fare_amount), 0) AS tip_rate
FROM source
GROUP BY pickup_date
ORDER BY pickup_date
```

**What you'll learn:** Pandas chunked reads, PostgreSQL DDL, dbt fundamentals, Docker networking

**Portfolio talking point:** "I built an end-to-end batch pipeline processing 40M NYC taxi trips using Python, PostgreSQL, and dbt. The dbt models apply Kimball-style dimensional modeling and include not_null and unique tests on all key columns."

---

### Project 2: Stock Price Pipeline (API → Cloud → Dashboard)

**Difficulty:** ⭐⭐ | **Time:** 1 week | **Resume Impact:** Good

**What you build:** Fetch daily stock prices from a public API, store in a cloud data warehouse, and build a simple dashboard.

**Architecture:**
```
Alpha Vantage / Yahoo Finance API (free)
    ↓ Python requests (daily cron or Airflow DAG)
AWS S3 (raw JSON landing zone)
    ↓ AWS Glue / Athena (schema discovery)
BigQuery or Redshift (warehouse)
    ↓ dbt (transform + test)
Looker Studio or Metabase (dashboard)
```

**Key features to include:**
- Paginated API fetching with retry logic
- Data validation (price > 0, volume > 0, no future dates)
- Airflow DAG with daily schedule and SLA alert
- dbt model for 30-day rolling average, daily returns

**What you'll learn:** REST API authentication, cloud storage, managed data warehouses, Airflow scheduling

---

### Project 3: Reddit Data Scraper + Analysis

**Difficulty:** ⭐⭐ | **Time:** 1 week | **Resume Impact:** Moderate

**What you build:** Collect posts from r/dataengineering using Reddit API, store in PostgreSQL, analyze trends (top tools mentioned, posting frequency, sentiment).

**What you'll learn:** OAuth 2.0 API authentication, JSON data handling, text analysis, PostgreSQL JSONB

---

## 4. Intermediate Projects (6-18 months)

### Project 4: Real-Time E-Commerce Analytics Platform

**Difficulty:** ⭐⭐⭐ | **Time:** 2-3 weeks | **Resume Impact:** Excellent

**What you build:** A streaming pipeline that simulates an e-commerce platform's event stream, processes it in real time, and feeds a live dashboard showing orders, revenue, and customer activity.

**Architecture:**
```
Event Generator (Python script)
    → simulates order_placed, payment_processed, item_shipped events
    → produces to Kafka

Apache Kafka (local Docker or Confluent Cloud free tier)
    ├── topic: ecommerce.orders.v1 (Avro + Schema Registry)
    ├── topic: ecommerce.payments.v1
    └── topic: ecommerce.shipments.v1

Apache Spark Structured Streaming (or Flink)
    → reads from Kafka
    → aggregates: revenue per minute, active customers, order status counts
    → writes to:

Delta Lake (S3 or local MinIO)
    ├── Bronze: raw events (append-only)
    ├── Silver: validated, deduplicated events
    └── Gold: fct_orders (incremental), dim_customers, revenue_by_hour

Apache Airflow
    → schedules daily dbt runs
    → monitors pipeline health

Apache Superset or Grafana
    → Live dashboard: revenue/min, orders/min, funnel conversion
```

**Event generator:**
```python
# event_generator.py
import json
import time
import random
from datetime import datetime
from kafka import KafkaProducer

producer = KafkaProducer(
    bootstrap_servers=["localhost:9092"],
    value_serializer=lambda v: json.dumps(v).encode("utf-8")
)

def generate_order():
    return {
        "event_type": "order_placed",
        "order_id": f"ORD-{random.randint(100000, 999999)}",
        "customer_id": f"CUST-{random.randint(1, 10000)}",
        "product_id": f"PROD-{random.randint(1, 500)}",
        "quantity": random.randint(1, 5),
        "unit_price": round(random.uniform(9.99, 299.99), 2),
        "timestamp": datetime.utcnow().isoformat(),
        "country": random.choice(["US", "UK", "DE", "FR", "JP", "AU"])
    }

while True:
    event = generate_order()
    producer.send("ecommerce.orders.v1", value=event)
    time.sleep(random.uniform(0.1, 0.5))  # 2-10 events per second
```

**Spark Structured Streaming consumer:**
```python
from pyspark.sql import SparkSession
from pyspark.sql import functions as F
from pyspark.sql.types import StructType, StringType, DoubleType, IntegerType

spark = SparkSession.builder \
    .appName("ecommerce-streaming") \
    .config("spark.jars.packages", "org.apache.spark:spark-sql-kafka-0-10_2.12:3.5.0") \
    .getOrCreate()

schema = StructType() \
    .add("order_id", StringType()) \
    .add("customer_id", StringType()) \
    .add("product_id", StringType()) \
    .add("quantity", IntegerType()) \
    .add("unit_price", DoubleType()) \
    .add("timestamp", StringType()) \
    .add("country", StringType())

raw_stream = spark.readStream \
    .format("kafka") \
    .option("kafka.bootstrap.servers", "localhost:9092") \
    .option("subscribe", "ecommerce.orders.v1") \
    .load()

parsed = raw_stream \
    .select(F.from_json(F.col("value").cast("string"), schema).alias("data")) \
    .select("data.*") \
    .withColumn("revenue", F.col("quantity") * F.col("unit_price")) \
    .withColumn("event_timestamp", F.to_timestamp("timestamp"))

# Write to Delta Lake Bronze layer
query = parsed.writeStream \
    .format("delta") \
    .option("checkpointLocation", "/delta/checkpoints/bronze_orders") \
    .outputMode("append") \
    .start("/delta/bronze/orders/")

query.awaitTermination()
```

**Portfolio talking point:** "Built a real-time e-commerce analytics pipeline using Kafka, Spark Structured Streaming, and Delta Lake (Medallion Architecture). The pipeline processes 10+ events/second with end-to-end latency under 30 seconds. Data is validated, deduplicated, and available in a live Superset dashboard."

---

### Project 5: GitHub Activity Analytics

**Difficulty:** ⭐⭐⭐ | **Time:** 2-3 weeks | **Resume Impact:** Excellent

**What you build:** Ingest GitHub Archive data (public, 5TB+), run analytics to identify trending repositories, active contributors, language popularity trends.

**Architecture:**
```
GitHub Archive (GCS public bucket: gs://githubarchive)
    ↓ dlt or custom ingestion script
BigQuery (data warehouse — BigQuery has a public dataset for this!)
    ↓ dbt (staging → dimensional model)
├── stg_github__events.sql
├── dim_repos.sql
├── dim_users.sql
└── fct_daily_activity.sql
    ↓
Looker Studio (Google's free BI — connects directly to BigQuery)
```

**Key analytics to build:**
- Top 10 fastest-growing repos by stars (week-over-week)
- Language popularity trends by quarter
- Most active contributors by repo
- PR merge time distribution

**What you'll learn:** BigQuery (partitioned tables, clustering, BQML), dbt on BigQuery, handling multi-TB datasets efficiently with partition pruning

**Cost estimate:** BigQuery free tier includes 10GB storage + 1TB queries/month. A well-partitioned query on GitHub Archive runs < 1GB → free.

---

### Project 6: CDC Pipeline with Debezium

**Difficulty:** ⭐⭐⭐ | **Time:** 2 weeks | **Resume Impact:** Very Good

**What you build:** Stream every INSERT/UPDATE/DELETE from a PostgreSQL database into a data lake using CDC, without modifying the source application.

**Architecture:**
```
PostgreSQL (simulated OLTP app)
    ↓ Debezium (CDC from WAL / pg_wal)
Apache Kafka
    ├── topic: db.public.orders (change events)
    └── topic: db.public.customers (change events)
    ↓ Kafka Connect (S3 Sink Connector)
S3 / MinIO (change events as Avro)
    ↓ Spark (merge into Delta Lake)
Delta Lake (SCD Type 2 — full history of every change)
    ↓ dbt (build dimensional model)
Analytical Query Layer
```

**docker-compose.yml:**
```yaml
version: '3'
services:
  postgres:
    image: debezium/postgres:15
    environment:
      POSTGRES_PASSWORD: password
      POSTGRES_DB: ecommerce
    ports: ["5432:5432"]
    command: postgres -c wal_level=logical

  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    depends_on: [zookeeper]
    environment:
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
    ports: ["9092:9092"]

  kafka-connect:
    image: debezium/connect:2.4
    depends_on: [kafka, postgres]
    environment:
      BOOTSTRAP_SERVERS: kafka:9092
      GROUP_ID: 1
      CONFIG_STORAGE_TOPIC: connect_configs
      OFFSET_STORAGE_TOPIC: connect_offsets
    ports: ["8083:8083"]
```

**What you'll learn:** Database CDC, Kafka Connect ecosystem, Avro serialization, SCD Type 2 implementation

---

### Project 7: dbt + BigQuery + Looker Studio End-to-End

**Difficulty:** ⭐⭐⭐ | **Time:** 2 weeks | **Resume Impact:** Very Good

**What you build:** A complete analytics engineering project using a public dataset (NYC Taxi or similar), following dbt best practices: sources, staging, intermediate, marts, tests, documentation.

**dbt project structure:**
```
models/
├── staging/
│   ├── _sources.yml              (freshness checks, column descriptions)
│   ├── stg_taxi__trips.sql       (type casting, cleaning)
│   ├── stg_taxi__zones.sql
│   └── schema.yml                (not_null, unique, accepted_values tests)
├── intermediate/
│   └── int_trips__enriched.sql   (join trips + zones, compute derived fields)
└── marts/
    ├── core/
    │   ├── dim_locations.sql
    │   ├── dim_date.sql
    │   └── fct_trips.sql         (incremental, partitioned by pickup_date)
    └── finance/
        └── daily_revenue.sql     (business aggregate)

tests/
└── test_revenue_never_decreases.sql  (custom test)

analyses/
└── q4_2024_report.sql            (ad-hoc analysis saved in version control
```

---

## 5. Advanced Projects (18+ months)

### Project 8: Lakehouse Migration (Hadoop → Iceberg)

**Difficulty:** ⭐⭐⭐⭐⭐ | **Time:** 4-6 weeks | **Resume Impact:** Outstanding

**What you build:** Simulate a migration from a legacy Hadoop/Hive data lake to a modern Apache Iceberg-based lakehouse, including:
- Schema migration with zero downtime
- Historical data backfill
- Validation that results match
- Automated tests proving data equivalence

**Why it's impressive:** Every company with legacy infrastructure needs to do this. Demonstrating you can design and execute it sets you apart.

**Architecture:**
```
Legacy Layer:
HDFS (local) + Hive Metastore + Hive tables (ORC format)
    ↓ Migration layer (Spark + Iceberg)

Modern Layer:
MinIO (S3-compatible) + Apache Iceberg + Nessie Catalog
    ↓ Query engines: Trino (interactive) + Spark (batch)
    ↓ dbt (transformation)

Validation Layer:
Great Expectations: row counts + aggregate checksums match
lakeFS: version-controlled data lake (git branch for migration)
```

**Key migration script:**
```python
# migrate_table.py
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .config("spark.sql.extensions", "org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions") \
    .config("spark.sql.catalog.iceberg", "org.apache.iceberg.spark.SparkCatalog") \
    .config("spark.sql.catalog.iceberg.type", "hadoop") \
    .config("spark.sql.catalog.iceberg.warehouse", "s3a://lakehouse/") \
    .getOrCreate()

def migrate_table(hive_db: str, hive_table: str, iceberg_table: str, partition_col: str):
    """Migrate a Hive ORC table to Iceberg Parquet with validation."""
    
    # Read legacy Hive table
    legacy_df = spark.table(f"{hive_db}.{hive_table}")
    
    # Create Iceberg table with same schema
    spark.sql(f"""
        CREATE TABLE IF NOT EXISTS iceberg.{iceberg_table}
        USING iceberg
        PARTITIONED BY ({partition_col})
        AS SELECT * FROM {hive_db}.{hive_table}
        WHERE 1=0  -- create empty table with schema
    """)
    
    # Backfill by partition (resumable)
    partitions = legacy_df.select(partition_col).distinct().collect()
    for row in partitions:
        partition_val = row[0]
        partition_df = legacy_df.filter(f"{partition_col} = '{partition_val}'")
        
        partition_df.writeTo(f"iceberg.{iceberg_table}") \
            .option("mergeSchema", "true") \
            .overwritePartitions()
        
        print(f"Migrated partition {partition_col}={partition_val}: {partition_df.count()} rows")
    
    # Validate: counts must match
    legacy_count = legacy_df.count()
    iceberg_count = spark.table(f"iceberg.{iceberg_table}").count()
    assert legacy_count == iceberg_count, \
        f"Count mismatch! Hive: {legacy_count}, Iceberg: {iceberg_count}"
    print(f"Migration validated: {legacy_count:,} rows match")

migrate_table("legacy_db", "orders", "main.orders", "order_date")
```

---

### Project 9: Multi-Tenant Data Platform

**Difficulty:** ⭐⭐⭐⭐⭐ | **Time:** 6-8 weeks | **Resume Impact:** Outstanding

**What you build:** A data platform that supports multiple teams/business units, each with their own:
- Isolated data (team A can't see team B's data)
- Compute quotas (team A's heavy query doesn't starve team B)
- Data catalog (each team sees their assets)
- dbt project (with shared macros)

**Architecture:**
```
Data Sources (multiple: CRM, ERP, marketing tools)
    ↓ Airbyte (per-tenant connector groups)

S3 (multi-tenant data lake)
├── s3://data-lake/team-finance/
├── s3://data-lake/team-marketing/
└── s3://data-lake/team-operations/

AWS Lake Formation (row/column level security)
    → Finance team: see revenue columns
    → Marketing team: see campaign data, not PII

BigQuery (multi-project setup)
├── project: finance-analytics
├── project: marketing-analytics
└── project: shared-dimensions

dbt (monorepo with multiple projects)
├── shared/          (shared macros, sources, dim_date)
├── finance/         (finance-specific models)
└── marketing/       (marketing-specific models)

OpenMetadata (unified data catalog across all teams)

Airflow (shared orchestration)
    → Per-team DAGs with separate pools (resource isolation)
```

---

### Project 10: ML Feature Store Pipeline

**Difficulty:** ⭐⭐⭐⭐⭐ | **Time:** 4-6 weeks | **Resume Impact:** Outstanding for ML-DE

**What you build:** A feature store that serves pre-computed features to ML models in both batch (training) and online (serving) contexts.

**Architecture:**
```
Raw Events (user activity, product catalog)
    ↓ Spark batch jobs (daily feature computation)

Feature Store:
├── Offline store: Delta Lake (for model training)
│   └── features: user_purchase_history, product_popularity, session_features
└── Online store: Redis (for real-time model serving)
    └── GET /features?user_id=123 → {"avg_order_value": 45.2, "days_since_last_order": 3}

Feature Registry:
├── Feature definitions (what, how computed, when refreshed)
├── Feature lineage (which features come from which sources)
└── Feature monitoring (drift detection, freshness)

Training pipeline:
    → Pull from offline store (point-in-time correct joins)
    → Train model (XGBoost, LightGBM)
    → MLflow experiment tracking

Serving pipeline:
    → Model loads features from online store (Redis, < 5ms)
    → Returns prediction + confidence score
```

---

## 6. Free Public Datasets

| Dataset | Size | Format | Access | Best For |
|---------|------|--------|--------|---------|
| **NYC Yellow Taxi Trips** | ~10GB/year | Parquet | [NYC TLC](https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page) | Batch, dbt, BigQuery |
| **GitHub Archive** | 5TB+ | JSON.gz | [GH Archive](https://www.gharchive.org/) | Large-scale analytics |
| **NYC Citibike** | ~5GB/year | CSV | [Citibike](https://citibikenyc.com/system-data) | Time series, geospatial |
| **Chicago Crime Data** | ~1.7GB | CSV | [City of Chicago](https://data.cityofchicago.org/) | SQL analysis, trends |
| **Open Weather Map** | Streaming | JSON API | [OpenWeather](https://openweathermap.org/api) | Real-time streaming project |
| **Reddit PushShift** | TBs | JSON | [PushShift](https://pushshift.io/) | NLP, text analytics |
| **Wikipedia Dumps** | ~20GB | XML | [Wikipedia](https://dumps.wikimedia.org/) | Graph analysis, NLP |
| **Common Crawl** | Petabytes | WARC | [Common Crawl](https://commoncrawl.org/) | Web-scale analytics |
| **NOAA Weather** | GBs | Various | [NOAA](https://www.noaa.gov/weather) | IoT simulation, streaming |
| **Stack Overflow Survey** | ~100MB | CSV | [SO](https://insights.stackoverflow.com/survey) | Data visualization |
| **Eventsim (Synthetic)** | Configurable | JSON stream | [GitHub](https://github.com/Interana/eventsim) | Streaming simulation |

---

## 7. Project Infrastructure Setup

### The Free Local Stack (No Cloud Cost)

Run your entire DE stack on your laptop for free:

```yaml
# docker-compose.yml — Full local DE stack
version: '3.8'

services:
  # PostgreSQL — operational DB and Airflow backend
  postgres:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: password
      POSTGRES_DB: airflow_db
    ports: ["5432:5432"]
    volumes:
      - postgres_data:/var/lib/postgresql/data

  # Apache Kafka — event streaming
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    depends_on: [zookeeper]
    ports: ["9092:9092"]
    environment:
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "true"

  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    depends_on: [kafka]
    ports: ["8080:8080"]
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:9092

  # MinIO — local S3-compatible object storage
  minio:
    image: minio/minio
    command: server /data --console-address ":9001"
    ports: ["9000:9000", "9001:9001"]
    environment:
      MINIO_ROOT_USER: minio
      MINIO_ROOT_PASSWORD: minio123
    volumes:
      - minio_data:/data

  # Airflow — pipeline orchestration
  airflow:
    image: apache/airflow:2.8.0
    depends_on: [postgres]
    ports: ["8088:8080"]
    environment:
      AIRFLOW__CORE__EXECUTOR: LocalExecutor
      AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql+psycopg2://postgres:password@postgres/airflow_db
      AIRFLOW__CORE__FERNET_KEY: "your-fernet-key-here"
    volumes:
      - ./dags:/opt/airflow/dags
    command: >
      bash -c "airflow db init && 
               airflow users create --username admin --password admin --firstname Admin --lastname User --role Admin --email admin@example.com &&
               airflow webserver & airflow scheduler"

  # Metabase — BI dashboards
  metabase:
    image: metabase/metabase
    ports: ["3000:3000"]
    depends_on: [postgres]

volumes:
  postgres_data:
  minio_data:
```

**Start everything:**
```bash
docker-compose up -d
# Kafka UI:    http://localhost:8080
# MinIO:       http://localhost:9001 (minio/minio123)
# Airflow:     http://localhost:8088 (admin/admin)
# Metabase:    http://localhost:3000
# PostgreSQL:  localhost:5432 (postgres/password)
```

### Free Cloud Services for Projects

| Service | Free Tier | What to Use It For |
|---------|-----------|-------------------|
| **BigQuery** | 10GB storage + 1TB queries/month | Data warehouse for any project |
| **GCS** | 5GB storage | Data lake for small projects |
| **Confluent Cloud** | $400 free credits | Managed Kafka |
| **dbt Cloud** | 1 developer seat free | Managed dbt runs |
| **MongoDB Atlas** | 512MB free cluster | Document store for projects |
| **Neon.tech** | Free PostgreSQL | Cloud Postgres (no Docker needed) |
| **Supabase** | 500MB PostgreSQL + 1GB storage | Postgres + APIs |

---

## 8. How to Present Projects to Employers

### The 3-Minute Verbal Summary

Practice explaining any project in 3 minutes:

```
1. Problem (30 seconds)
"I built a real-time e-commerce analytics pipeline. 
The business problem: analytics teams only saw yesterday's data.
I wanted to show how to get that latency down to under 1 minute."

2. Architecture (60 seconds)
"Events flow from a Python event generator into Kafka with Avro 
schemas validated by Schema Registry. Spark Structured Streaming 
reads the Kafka stream, applies quality checks, and writes to 
Delta Lake in a Medallion Architecture — Bronze, Silver, Gold layers.
Airflow orchestrates the daily dbt runs that build the dimensional
models in the Gold layer."

3. Key decisions and trade-offs (60 seconds)
"I chose Spark Structured Streaming over Flink because my team 
would already know Spark — one language for batch and streaming.
I chose Delta Lake over raw Parquet because I needed MERGE for 
the CDC updates, and time travel for debugging failed runs."

4. Result and what I learned (30 seconds)
"End-to-end latency is under 30 seconds. I learned a lot about
Kafka consumer lag monitoring — I set up a Prometheus alert when
lag exceeded 50,000 messages. Next time I'd add Schema Registry
from day one rather than retrofitting it."
```

### GitHub Repository Checklist

Before sharing a project repo:

- [ ] README explains the problem, architecture, tech stack, and how to run it
- [ ] `docker-compose up` works without errors (test on a clean machine)
- [ ] `.env.example` provided (never commit real credentials)
- [ ] Code is organized (not all in one 1,000-line file)
- [ ] At least one test (dbt test, pytest, or Great Expectations)
- [ ] Secrets are in `.gitignore` — no API keys in git history
- [ ] Architecture diagram (even an ASCII one is fine)
- [ ] Realistic data volume (not just 10 rows)

### LinkedIn Post Template

```
🚀 Just built [Project Name] — here's what I learned:

[1-2 sentence problem statement]

Architecture:
• [Tool 1]: [why you chose it]
• [Tool 2]: [why you chose it]
• [Tool 3]: [why you chose it]

Key challenges:
• [Challenge 1 and how you solved it]
• [Challenge 2 and how you solved it]

What surprised me:
• [Honest insight — something harder or different than expected]

The code is on GitHub 👇
[link]

#DataEngineering #[Tool] #[Tool] #OpenSource
```

---

## Project Roadmap Summary

```
Month 1-3 (Beginner):
  → Project 1: NYC Taxi Pipeline (Python + PostgreSQL + dbt)
  → Goal: Prove you can build end-to-end, write clean dbt models

Month 4-6 (Beginner→Intermediate):
  → Project 2: Stock Price API Pipeline (Airflow + cloud warehouse)
  → Goal: Add orchestration + cloud services

Month 7-12 (Intermediate):
  → Project 4: Real-Time E-Commerce (Kafka + Spark Streaming + Delta Lake)
  → Goal: Streaming pipeline on your resume

Month 13-18 (Intermediate→Advanced):
  → Project 6: CDC with Debezium (Postgres → Kafka → Delta Lake)
  → Goal: Demonstrate CDC, which every company needs

18+ months (Advanced):
  → Project 8 or 9 (Migration or Multi-Tenant Platform)
  → Goal: Senior-level architectural complexity

Always running:
  → GitHub: Push code weekly, even if it's WIP
  → LinkedIn: Post about each project (one post per project minimum)
  → dbt: Add tests to every model (demonstrates production-readiness mindset)
```

---

← [15 Communities](15-communities.md) | [Back to Home →](../README.md)
