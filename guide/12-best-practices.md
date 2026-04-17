# Best Practices in Data Engineering

**Author:** Mano Harsha Sappa | [LinkedIn](https://www.linkedin.com/in/manoharshasappa/) | [Portfolio](https://manoharshasappa.github.io/portfolio_ManoHarshaSappa/) | [Email](mailto:sappamanoharsha@gmail.com)

---

## Contents

1. [Pipeline Design Principles](#1-pipeline-design-principles)
2. [Data Quality & Reliability](#2-data-quality--reliability)
3. [Performance & Cost Optimization](#3-performance--cost-optimization)
4. [Security & Compliance](#4-security--compliance)
5. [Observability & Monitoring](#5-observability--monitoring)
6. [CI/CD for Data Pipelines](#6-cicd-for-data-pipelines)
7. [Schema & Data Contract Management](#7-schema--data-contract-management)
8. [SQL & dbt Best Practices](#8-sql--dbt-best-practices)
9. [Kafka Best Practices](#9-kafka-best-practices)
10. [Spark Best Practices](#10-spark-best-practices)
11. [Cloud Architecture Best Practices](#11-cloud-architecture-best-practices)
12. [Team & Process Best Practices](#12-team--process-best-practices)
13. [The 81 Platform Design Questions](#13-the-81-platform-design-questions)

---

## 1. Pipeline Design Principles

These are the principles that separate amateur pipelines from production-grade pipelines.

### Idempotency

An idempotent pipeline produces the same result no matter how many times it runs. This is the single most important property of a data pipeline.

**Why it matters:** Pipelines fail. They get re-run. If your pipeline isn't idempotent, re-running it will create duplicate data, wrong counts, or corrupted state.

```python
# BAD: Not idempotent — re-running inserts duplicate rows
def load_daily_orders(date):
    df = fetch_orders(date)
    db.execute("INSERT INTO orders SELECT * FROM staging_orders")
    # If this runs twice: doubled data

# GOOD: Idempotent — re-running deletes+replaces = same result
def load_daily_orders(date):
    df = fetch_orders(date)
    db.execute(f"DELETE FROM orders WHERE order_date = '{date}'")
    db.execute("INSERT INTO orders SELECT * FROM staging_orders")
    # If this runs twice: same result
    
# EVEN BETTER: MERGE (upsert) — handles both inserts and updates
db.execute(f"""
    MERGE INTO orders AS target
    USING staging_orders AS source
    ON target.order_id = source.order_id
    WHEN MATCHED THEN UPDATE SET ...
    WHEN NOT MATCHED THEN INSERT ...
""")
```

### Exactly-Once Semantics

In distributed systems, networks fail. Ensuring a message is processed exactly once is hard:

| Guarantee | Risk | When to Use |
|-----------|------|-------------|
| **At-most-once** | Data loss (miss events) | Non-critical analytics |
| **At-least-once** | Duplicates (process twice) | Acceptable with deduplication |
| **Exactly-once** | Complex but safe | Financial transactions, billing |

**Practical approach:** Use at-least-once delivery + idempotent writes. This is simpler than true exactly-once and delivers equivalent correctness.

### Backfilling

Design every pipeline to support historical backfill from day one.

```python
# Parameterize by date, not "yesterday"
# BAD
def run():
    date = datetime.now() - timedelta(1)  # Always yesterday
    process(date)

# GOOD
def run(date: str):
    process(date)

# Now you can backfill:
for date in date_range("2024-01-01", "2024-12-31"):
    run(date)
```

### Late-Arriving Data

Real-world data arrives late. An order placed at 11:58 PM might not reach your warehouse until 1:00 AM.

**Strategy 1: Watermarks** (stream processing)
```python
# Flink: tolerate events up to 10 minutes late
stream.assign_timestamps_and_watermarks(
    WatermarkStrategy
        .for_bounded_out_of_orderness(Duration.of_minutes(10))
        .with_timestamp_assigner(lambda event, _: event.event_time)
)
```

**Strategy 2: Lookback window** (batch processing)
```sql
-- Don't just process today's data
-- Look back 3 days to catch late-arriving events
WHERE event_date >= CURRENT_DATE - INTERVAL 3 DAYS
AND event_date < CURRENT_DATE
```

**Strategy 3: Reprocessing triggers** (Lambda approach)
- Batch job runs daily for T-1 (yesterday)
- Another job runs T-7 (one week ago) to catch late data
- Final reconciliation job runs T-30

---

## 2. Data Quality & Reliability

### The Data Quality Dimensions

| Dimension | Definition | How to Measure |
|-----------|------------|----------------|
| **Completeness** | Are all expected records present? | Row count vs source |
| **Accuracy** | Do values match reality? | Spot checks, cross-validation |
| **Consistency** | Do related tables agree? | Referential integrity checks |
| **Timeliness** | Is data fresh enough? | Max event_time vs NOW() |
| **Uniqueness** | Are there duplicates? | COUNT(*) vs COUNT(DISTINCT id) |
| **Validity** | Are values in valid ranges? | Min/max, enum validation |

### Defensive Data Loading

```python
def load_with_validation(df, target_table, expected_row_range=(1000, 10_000_000)):
    """Never load data blindly — validate before every load."""
    row_count = df.count()
    
    # 1. Row count sanity check
    if not (expected_row_range[0] <= row_count <= expected_row_range[1]):
        raise ValueError(
            f"Row count {row_count} outside expected range {expected_row_range}"
        )
    
    # 2. Null check on primary key
    null_pks = df.filter(df.id.isNull()).count()
    if null_pks > 0:
        raise ValueError(f"Found {null_pks} null primary keys")
    
    # 3. Duplicate check
    duplicates = row_count - df.dropDuplicates(["id"]).count()
    if duplicates > 0:
        raise ValueError(f"Found {duplicates} duplicate IDs")
    
    # 4. All checks passed — load
    df.write.mode("overwrite").saveAsTable(target_table)
    print(f"Loaded {row_count:,} rows to {target_table}")
```

### The Testing Pyramid for Data

```
        /\
       /  \    Integration Tests (expensive, slow)
      /    \   - Run dbt tests against real warehouse
     /      \  - End-to-end pipeline smoke tests
    /--------\
   /          \ Unit Tests (fast, cheap)
  /            \ - Test individual transformations
 /              \ - Test business logic functions
/----------------\ Contract Tests (always-on)
                   - Schema validation on ingestion
                   - Row count anomaly detection
```

### Data SLAs (Service Level Agreements)

Define SLAs for your data and monitor against them:

```yaml
# Example data SLA definitions
tables:
  - name: fct_orders
    sla:
      freshness_hours: 4        # Data must be < 4 hours old
      min_daily_rows: 10000     # At least 10k orders/day
      null_rate_threshold: 0.01 # Less than 1% nulls on key columns
      
  - name: dim_customers
    sla:
      freshness_hours: 24       # Dimensions can be 24h stale
      uniqueness_columns:
        - customer_id            # Must be unique
```

---

## 3. Performance & Cost Optimization

### The Golden Rule

> Optimize only after you've measured. Never optimize on assumptions.

Profile first:
```sql
-- BigQuery: check bytes scanned before running
-- GCP charges $5/TB scanned — always check first!
SELECT *
FROM orders
WHERE order_date = '2024-01-15'
-- Bytes processed: 2.5 GB (if partitioned on order_date)
-- Bytes processed: 500 GB (if NOT partitioned)
```

### Partitioning

Partitioning is the single highest-ROI optimization for analytical queries.

```sql
-- BigQuery: partitioned + clustered table (correct approach)
CREATE TABLE orders
PARTITION BY DATE(order_date)    -- Partition prunes entire partitions
CLUSTER BY customer_id, status   -- Cluster prunes within partitions
AS
SELECT * FROM raw_orders;

-- Query with partition filter: scans 1/365th of the data
SELECT * FROM orders
WHERE order_date = '2024-01-15'  -- Only reads Jan 15 partition
AND customer_id = 'C123'         -- Clustering further reduces scan
```

**Partition strategy by use case:**
```
Daily analytics jobs:     PARTITION BY date
User-specific queries:    PARTITION BY user_id % 100 (hash partitioning)
Geographic queries:       PARTITION BY country
Archive/hot/cold:         PARTITION BY year (yearly compaction)
```

### Spark Performance Tuning

**Avoid shuffles where possible:**
```python
# BAD: groupBy causes shuffle (expensive network transfer)
df.groupBy("user_id").agg(F.sum("revenue"))

# If both DFs are large — use broadcast join to avoid shuffle
from pyspark.sql.functions import broadcast
small_df = spark.table("dim_products")  # < 100MB
large_df = spark.table("fct_orders")    # Billions of rows

# BAD: Sort-merge join (shuffle both sides)
result = large_df.join(small_df, "product_id")

# GOOD: Broadcast join (no shuffle on large_df)
result = large_df.join(broadcast(small_df), "product_id")
```

**Partition your Spark DataFrame appropriately:**
```python
# Read from S3: coalesce or repartition based on data size
df = spark.read.parquet("s3://bucket/orders/")

# Too many small files? Coalesce (no shuffle)
df = df.coalesce(200)

# Need even distribution? Repartition (with shuffle)
df = df.repartition(200, "customer_id")

# Write with the right number of output files
df.write.partitionBy("date").parquet("s3://bucket/output/")
```

**Caching strategy:**
```python
# Cache only DataFrames that are reused multiple times
base_orders = spark.table("raw_orders").filter("year = 2024").cache()
base_orders.count()  # Materialize the cache

# Use the cache
revenue_by_customer = base_orders.groupBy("customer_id").agg(...)
revenue_by_product  = base_orders.groupBy("product_id").agg(...)

# Release when done
base_orders.unpersist()
```

### Cost Optimization Checklist

**Storage:**
- [ ] Use Parquet/ORC (5-10x smaller than CSV)
- [ ] Apply compression (Snappy for hot data, Zstd for archives)
- [ ] Set S3 lifecycle rules (S3-IA after 30 days, Glacier after 90 days)
- [ ] Delete old data (how long do you actually need 5-year-old clickstream?)
- [ ] Compact small files into large ones (avoid S3 API overhead)

**Compute:**
- [ ] Use spot/preemptible instances for batch (70-90% savings)
- [ ] Right-size clusters (don't use r6g.8xlarge for a 1GB dataset)
- [ ] Schedule batch jobs during off-peak hours (lower on-demand price)
- [ ] Use auto-scaling (scale down idle clusters)
- [ ] Cache frequently-accessed data in Alluxio or Redis

**Query:**
- [ ] Always use partition filters in WHERE clauses
- [ ] Avoid `SELECT *` — read only the columns you need
- [ ] Use approximate aggregations where exactness isn't required (COUNT APPROX)
- [ ] Materialize frequently-run expensive queries as views or pre-aggregations
- [ ] Monitor query cost in BigQuery/Redshift before running

---

## 4. Security & Compliance

### The Principle of Least Privilege

Grant only the permissions needed for the task. Never grant admin to an application.

```sql
-- BAD: Give the ETL service full admin access
GRANT ALL PRIVILEGES ON *.* TO 'etl_service'@'%';

-- GOOD: Grant only what's needed
CREATE ROLE etl_pipeline_role;
GRANT SELECT ON raw_schema.* TO etl_pipeline_role;
GRANT INSERT, UPDATE ON staging_schema.* TO etl_pipeline_role;
GRANT SELECT ON staging_schema.* TO etl_pipeline_role;
-- The ETL service cannot DROP tables, DELETE from raw, or touch prod

CREATE USER 'etl_service'@'%';
GRANT etl_pipeline_role TO 'etl_service'@'%';
```

### Secret Management

Never hardcode credentials. Use secret management services.

```python
# BAD: Credentials in code (never do this)
conn = psycopg2.connect(
    host="prod-db.example.com",
    password="SuperSecret123!"  # This will end up in git
)

# GOOD: Credentials from environment variables
import os
conn = psycopg2.connect(
    host=os.environ["DB_HOST"],
    password=os.environ["DB_PASSWORD"]
)

# BEST: Credentials from AWS Secrets Manager (auto-rotates)
import boto3, json

def get_secret(secret_name):
    client = boto3.client("secretsmanager", region_name="us-east-1")
    response = client.get_secret_value(SecretId=secret_name)
    return json.loads(response["SecretString"])

creds = get_secret("prod/database/credentials")
conn = psycopg2.connect(
    host=creds["host"],
    password=creds["password"]
)
```

### PII and GDPR Compliance

**GDPR rights that affect data engineering:**

| Right | What It Means for DE |
|-------|---------------------|
| **Right to be forgotten** | Must be able to delete all data for a user across ALL systems |
| **Right to access** | Must be able to export all data for a user |
| **Data minimization** | Only collect what you need |
| **Purpose limitation** | Don't use data for purposes other than what was consented |
| **Data residency** | EU user data must stay in EU regions |

**Implementing right to be forgotten:**
```python
def delete_user_data(user_id: str):
    """GDPR Article 17: Right to erasure."""
    
    # 1. Delete from operational database
    db.execute("DELETE FROM users WHERE user_id = %s", (user_id,))
    db.execute("DELETE FROM user_events WHERE user_id = %s", (user_id,))
    
    # 2. Mark as deleted in data warehouse (physical delete is hard in partitioned tables)
    warehouse.execute(f"""
        UPDATE fct_orders
        SET customer_email = 'REDACTED',
            customer_name = 'REDACTED',
            gdpr_deleted_at = CURRENT_TIMESTAMP
        WHERE user_id = '{user_id}'
    """)
    
    # 3. Remove from ML training data and retrain
    delete_from_feature_store(user_id)
    
    # 4. Delete from data lake (Iceberg DELETE FROM)
    iceberg.execute(f"DELETE FROM events WHERE user_id = '{user_id}'")
    
    # 5. Log the deletion for audit trail
    audit_log.record(f"GDPR deletion completed for user {user_id}")
```

**PII detection and masking:**
```python
import hashlib
import re

def mask_pii(value: str, pii_type: str) -> str:
    """Mask PII before storing in data lake."""
    if pii_type == "email":
        return hashlib.sha256(value.lower().encode()).hexdigest()[:16]
    elif pii_type == "phone":
        return re.sub(r'\d', 'X', value[:-4]) + value[-4:]
    elif pii_type == "ssn":
        return "XXX-XX-" + value[-4:]
    elif pii_type == "credit_card":
        return "XXXX-XXXX-XXXX-" + value[-4:]
    return value

# Never store raw email in the data lake
user_record["email_hash"] = mask_pii(user_record["email"], "email")
del user_record["email"]  # Remove raw PII
```

### Encryption

```python
# Data at rest: S3 server-side encryption (SSE-KMS)
s3.put_object(
    Bucket="data-lake",
    Key="orders/2024/orders.parquet",
    Body=parquet_bytes,
    ServerSideEncryption="aws:kms",
    SSEKMSKeyId="arn:aws:kms:us-east-1:123456789:key/..."
)

# Data in transit: always use TLS
# In Kafka producer:
producer = KafkaProducer(
    bootstrap_servers=["kafka:9093"],  # 9093 = TLS port
    security_protocol="SSL",
    ssl_cafile="/etc/kafka/ca-cert",
    ssl_certfile="/etc/kafka/client-cert",
    ssl_keyfile="/etc/kafka/client-key"
)
```

---

## 5. Observability & Monitoring

### The Four Golden Signals for Data Pipelines

| Signal | What to Measure | Alert If |
|--------|-----------------|---------|
| **Latency** | Time from event to availability in warehouse | > SLA threshold |
| **Traffic** | Row count, event rate, bytes processed | < 50% or > 200% of normal |
| **Errors** | Failed tasks, exceptions, validation failures | Any failure in critical pipelines |
| **Saturation** | Queue depth, consumer lag, disk usage | Consumer lag > 1M messages |

### Airflow Monitoring

```python
from airflow.models import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime, timedelta

default_args = {
    "owner": "data-engineering",
    "retries": 3,
    "retry_delay": timedelta(minutes=5),
    "email": ["data-alerts@company.com"],
    "email_on_failure": True,          # Alert immediately on failure
    "email_on_retry": False,
    "sla": timedelta(hours=4),          # Alert if DAG hasn't completed in 4h
}

with DAG(
    "daily_orders_pipeline",
    default_args=default_args,
    schedule_interval="0 6 * * *",     # 6 AM daily
    catchup=False,
    tags=["orders", "critical"]
) as dag:
    pass
```

### Data Freshness Monitoring

```sql
-- Check freshness of all critical tables (run every 15 minutes)
WITH table_freshness AS (
    SELECT 'fct_orders' AS table_name,
           MAX(order_date) AS latest_date,
           TIMESTAMP_DIFF(CURRENT_TIMESTAMP(), MAX(updated_at), HOUR) AS hours_stale
    FROM fct_orders
    
    UNION ALL
    
    SELECT 'dim_customers',
           MAX(updated_date),
           TIMESTAMP_DIFF(CURRENT_TIMESTAMP(), MAX(updated_at), HOUR)
    FROM dim_customers
)
SELECT *
FROM table_freshness
WHERE hours_stale > 6  -- Alert if any table is > 6 hours stale
ORDER BY hours_stale DESC;
```

### Kafka Consumer Lag Monitoring

```python
# Monitor consumer lag — critical for streaming pipelines
from kafka.admin import KafkaAdminClient
from kafka import KafkaConsumer

def get_consumer_lag(group_id, topic):
    consumer = KafkaConsumer(
        bootstrap_servers=["kafka:9092"],
        group_id=group_id,
        enable_auto_commit=False
    )
    
    # Get current consumer positions and end offsets
    partitions = consumer.partitions_for_topic(topic)
    topic_partitions = [TopicPartition(topic, p) for p in partitions]
    
    end_offsets = consumer.end_offsets(topic_partitions)
    current_positions = {tp: consumer.position(tp) for tp in topic_partitions}
    
    total_lag = sum(
        end_offsets[tp] - current_positions[tp]
        for tp in topic_partitions
    )
    
    if total_lag > 1_000_000:
        send_alert(f"HIGH KAFKA LAG: {group_id} is {total_lag:,} messages behind!")
    
    return total_lag
```

### Pipeline Lineage

Track which source data feeds which downstream tables. When upstream changes, know what breaks.

```
Raw (S3)
├── stg_orders (staging model)
│   ├── fct_orders (fact table)
│   │   ├── daily_revenue (aggregate)
│   │   │   └── Executive Dashboard ← if fct_orders breaks, THIS breaks
│   │   └── customer_ltv
│   └── dim_customers
│       └── customer_segments_report
```

With DataHub or OpenLineage, this lineage is tracked automatically.

---

## 6. CI/CD for Data Pipelines

### The dbt CI/CD Pattern

```yaml
# .github/workflows/dbt_ci.yml
name: dbt CI

on:
  pull_request:
    branches: [main]
    paths: ["dbt/**"]

jobs:
  dbt_ci:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Install dbt
        run: pip install dbt-bigquery
      
      - name: dbt compile (syntax check)
        run: dbt compile --target ci
      
      - name: dbt test on modified models only
        run: |
          # Use state-based CI: only run what changed
          dbt build \
            --select state:modified+ \
            --defer \
            --state ./prod_artifacts \
            --target ci
        env:
          DBT_BIGQUERY_PROJECT: ${{ secrets.GCP_PROJECT_CI }}
          
      - name: Upload dbt artifacts
        uses: actions/upload-artifact@v3
        with:
          name: dbt-artifacts
          path: target/manifest.json
```

**State-based CI explained:**
- `state:modified+` — only run models that changed AND their downstream dependents
- `--defer` — use production data for unchanged upstream models (no full rebuild)
- `--state ./prod_artifacts` — compare against last production manifest

This reduces CI runtime from 2 hours to 5 minutes for typical PRs.

### Automated Data Pipeline Testing

```python
# tests/test_daily_orders_pipeline.py
import pytest
from unittest.mock import patch, MagicMock
from pipeline.daily_orders import extract_orders, transform_orders, load_orders

@pytest.fixture
def sample_orders():
    return [
        {"order_id": "O1", "customer_id": "C1", "revenue": 100.0, "status": "completed"},
        {"order_id": "O2", "customer_id": "C2", "revenue": 250.5, "status": "pending"},
    ]

def test_transform_orders_calculates_totals(sample_orders):
    result = transform_orders(sample_orders)
    assert len(result) == 2
    assert result[0]["revenue_cents"] == 10000  # 100.0 * 100

def test_transform_orders_filters_cancelled(sample_orders):
    sample_orders.append(
        {"order_id": "O3", "customer_id": "C3", "revenue": 75.0, "status": "cancelled"}
    )
    result = transform_orders(sample_orders)
    assert len(result) == 2  # Cancelled orders excluded

def test_load_orders_is_idempotent(sample_orders, mock_db):
    """Running load twice should produce the same result."""
    load_orders(sample_orders, date="2024-01-15")
    load_orders(sample_orders, date="2024-01-15")
    
    count = mock_db.execute("SELECT COUNT(*) FROM orders WHERE date = '2024-01-15'")
    assert count == 2  # Not 4 (idempotent)
```

### Infrastructure as Code Best Practices

```hcl
# terraform/environments/prod/main.tf

# Use remote state (never local)
terraform {
  backend "s3" {
    bucket = "company-terraform-state"
    key    = "prod/data-platform/terraform.tfstate"
    region = "us-east-1"
    
    # State locking with DynamoDB (prevents concurrent applies)
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}

# Use variables, not hardcoded values
variable "environment" {
  description = "Deployment environment"
  type        = string
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

# Tag everything for cost attribution
locals {
  common_tags = {
    Environment = var.environment
    Team        = "data-engineering"
    ManagedBy   = "terraform"
    CostCenter  = "data-platform"
  }
}
```

---

## 7. Schema & Data Contract Management

### What is a Data Contract?

A data contract is an agreement between data producers and consumers about:
- Schema (column names, types, constraints)
- Freshness (when data will be available)
- Quality (expected null rates, value ranges)
- Semantics (what `revenue` means — gross? net? before tax?)

**Without data contracts:** A backend engineer renames a column from `user_id` to `userId` and breaks 15 downstream pipelines overnight.

**With data contracts:** The change is caught in CI before it reaches production.

### Implementing Schema Validation at Ingestion

```python
from pydantic import BaseModel, validator
from typing import Optional
from decimal import Decimal

class OrderEvent(BaseModel):
    """Data contract for order events from the e-commerce platform."""
    order_id: str
    customer_id: str
    revenue: Decimal
    currency: str
    status: str
    created_at: str
    
    @validator("revenue")
    def revenue_must_be_positive(cls, v):
        if v < 0:
            raise ValueError("Revenue cannot be negative")
        return v
    
    @validator("status")
    def status_must_be_valid(cls, v):
        valid_statuses = {"pending", "processing", "shipped", "delivered", "cancelled"}
        if v not in valid_statuses:
            raise ValueError(f"Invalid status: {v}. Must be one of {valid_statuses}")
        return v
    
    @validator("currency")
    def currency_must_be_iso(cls, v):
        if len(v) != 3:
            raise ValueError(f"Currency must be 3-letter ISO code, got: {v}")
        return v.upper()

# In your ingestion pipeline
def ingest_order(raw_event: dict):
    try:
        validated = OrderEvent(**raw_event)  # Validates at ingestion
        return validated.dict()
    except ValidationError as e:
        # Send to dead-letter queue, don't lose the event
        send_to_dlq(raw_event, str(e))
        raise
```

### Avro Schema Registry Pattern

```python
# Schema Registry enforces compatibility between producer and consumer
from confluent_kafka.schema_registry import SchemaRegistryClient
from confluent_kafka.schema_registry.avro import AvroSerializer

schema_registry_conf = {"url": "http://schema-registry:8081"}
schema_registry_client = SchemaRegistryClient(schema_registry_conf)

order_schema = """
{
    "type": "record",
    "name": "Order",
    "namespace": "com.company.ecommerce",
    "fields": [
        {"name": "order_id", "type": "string"},
        {"name": "customer_id", "type": "string"},
        {"name": "revenue", "type": "double"},
        {"name": "status", "type": {
            "type": "enum",
            "name": "OrderStatus",
            "symbols": ["pending", "shipped", "delivered", "cancelled"]
        }},
        {"name": "created_at", "type": "long", "logicalType": "timestamp-millis"}
    ]
}
"""

# Schema Registry enforces BACKWARD compatibility by default
# Breaking change (remove field) → CI fails
# Safe change (add optional field) → CI passes
avro_serializer = AvroSerializer(schema_registry_client, order_schema)
```

---

## 8. SQL & dbt Best Practices

### Write Readable SQL

```sql
-- BAD: Hard to read and maintain
SELECT a.id,b.name,c.total FROM orders a JOIN customers b ON a.cid=b.id JOIN (SELECT order_id,SUM(rev) total FROM items GROUP BY 1) c ON a.id=c.order_id WHERE a.dt>'2024-01-01' AND b.country='US';

-- GOOD: Readable SQL with clear structure
WITH order_items_summary AS (
    SELECT
        order_id,
        SUM(revenue) AS total_revenue
    FROM order_items
    GROUP BY order_id
),

us_customers AS (
    SELECT id, name
    FROM customers
    WHERE country = 'US'
)

SELECT
    orders.id        AS order_id,
    customers.name   AS customer_name,
    summary.total_revenue
FROM orders
INNER JOIN us_customers AS customers
    ON orders.customer_id = customers.id
INNER JOIN order_items_summary AS summary
    ON orders.id = summary.order_id
WHERE orders.created_date > '2024-01-01'
ORDER BY summary.total_revenue DESC
```

### dbt Naming Conventions

```
Staging models:   stg_{source}__{entity}.sql
                  stg_stripe__charges.sql
                  stg_postgres__users.sql

Intermediate:     int_{description}.sql
                  int_orders__enriched.sql
                  int_sessions__unioned.sql

Fact tables:      fct_{entity}.sql
                  fct_orders.sql
                  fct_web_sessions.sql

Dimension tables: dim_{entity}.sql
                  dim_customers.sql
                  dim_products.sql

Aggregates/Marts: {subject}_{grain}.sql
                  daily_revenue_summary.sql
                  customer_ltv.sql
```

### dbt Model Configuration

```sql
-- fct_orders.sql
{{
    config(
        materialized='incremental',
        unique_key='order_id',
        partition_by={
            "field": "order_date",
            "data_type": "date",
            "granularity": "day"
        },
        cluster_by=["customer_id"],
        on_schema_change='fail',    -- Fail if schema changes unexpectedly
        tags=['daily', 'critical']  -- Used for selecting which models to run
    )
}}

WITH source AS (
    SELECT *
    FROM {{ ref('stg_ecommerce__orders') }}
    {% if is_incremental() %}
    WHERE order_date >= (SELECT MAX(order_date) - INTERVAL 3 DAY FROM {{ this }})
    -- Look back 3 days for late-arriving orders
    {% endif %}
)

SELECT
    order_id,
    customer_id,
    order_date,
    revenue,
    CURRENT_TIMESTAMP() AS _loaded_at
FROM source
```

### SQL Performance Anti-Patterns

```sql
-- ANTI-PATTERN 1: SELECT *
SELECT * FROM fct_orders;  -- Reads ALL columns including large JSONs
-- FIX: Name the columns you need
SELECT order_id, customer_id, revenue, order_date FROM fct_orders;

-- ANTI-PATTERN 2: NOT IN with NULLs
SELECT * FROM orders WHERE customer_id NOT IN (SELECT id FROM deleted_customers);
-- If deleted_customers has ANY NULL, returns 0 rows!
-- FIX: Use NOT EXISTS
SELECT * FROM orders o
WHERE NOT EXISTS (
    SELECT 1 FROM deleted_customers dc WHERE dc.id = o.customer_id
);

-- ANTI-PATTERN 3: Functions on indexed/partitioned columns
WHERE DATE(created_at) = '2024-01-15'  -- Function prevents partition pruning
-- FIX:
WHERE created_at >= '2024-01-15' AND created_at < '2024-01-16'

-- ANTI-PATTERN 4: DISTINCT to fix duplicates
SELECT DISTINCT customer_id, order_date FROM orders;
-- DISTINCT is a symptom — fix the upstream duplication
-- FIX: Find why you have duplicates and remove them at source

-- ANTI-PATTERN 5: CROSS JOIN with no filter
SELECT * FROM orders, customers;  -- Cartesian product: millions × millions
-- FIX: Always have a JOIN condition
SELECT * FROM orders JOIN customers ON orders.customer_id = customers.id;
```

---

## 9. Kafka Best Practices

### Topic Design

```
# Naming convention: {environment}.{domain}.{entity}.{version}
prod.ecommerce.orders.v1
prod.ecommerce.order-items.v1
staging.analytics.user-events.v2

# Partition count: plan for your target consumer parallelism
# Rule of thumb: target_throughput_mb_per_sec / partition_throughput_mb_per_sec
# Example: 100 MB/s target, 10 MB/s per partition → 10 partitions
# Once set, increasing partitions breaks ordering guarantees

# Replication factor: always 3 in production
# min.insync.replicas: 2 (tolerate 1 broker failure)
```

### Producer Best Practices

```python
producer = KafkaProducer(
    bootstrap_servers=["kafka:9092"],
    
    # Durability settings
    acks="all",                          # Wait for all ISR replicas
    retries=10,                          # Retry on transient failures
    retry_backoff_ms=100,                # Exponential backoff
    max_in_flight_requests_per_connection=1,  # Preserve order during retries
    
    # Throughput settings  
    compression_type="lz4",              # Fast compression
    batch_size=65536,                    # 64KB batches
    linger_ms=5,                         # Wait 5ms to fill batch
    
    # Serialization
    value_serializer=lambda v: json.dumps(v).encode("utf-8"),
    key_serializer=lambda k: k.encode("utf-8")
)

# Always include a key for ordering guarantees
producer.send(
    "prod.ecommerce.orders.v1",
    key=order["customer_id"],  # All orders for a customer go to same partition
    value=order
)
```

### Consumer Best Practices

```python
consumer = KafkaConsumer(
    "prod.ecommerce.orders.v1",
    bootstrap_servers=["kafka:9092"],
    group_id="order-analytics-consumer",
    
    # Offset management
    auto_offset_reset="earliest",        # Start from beginning if no offset
    enable_auto_commit=False,            # Manual commit (safer)
    
    # Performance
    max_poll_records=500,                # Process 500 records per poll
    fetch_min_bytes=1048576,             # 1MB minimum fetch
    fetch_max_wait_ms=500,               # Wait up to 500ms for 1MB
)

for message_batch in consumer:
    messages = consumer.poll(timeout_ms=1000)
    for tp, msgs in messages.items():
        for msg in msgs:
            process_order(msg.value)
    
    # Only commit after successful processing
    consumer.commit()
```

### Consumer Lag is Your SLO

Set a consumer lag SLO and alert on it:
```
Consumer lag = End offset - Consumer offset

Target: lag < 10,000 messages (< 1 minute behind at normal throughput)
Warning: lag > 100,000 messages
Critical: lag > 1,000,000 messages → scale out consumers or fix processing bottleneck
```

---

## 10. Spark Best Practices

### Spark Configuration for Production

```python
spark = (
    SparkSession.builder
    .appName("daily-orders-pipeline")
    
    # Memory tuning
    .config("spark.executor.memory", "8g")
    .config("spark.executor.memoryOverhead", "2g")  # For Python/JVM overhead
    .config("spark.driver.memory", "4g")
    
    # Parallelism
    .config("spark.sql.shuffle.partitions", "200")  # Default 200 (tune per job)
    .config("spark.default.parallelism", "200")
    
    # Adaptive Query Execution (Spark 3.x game-changer)
    .config("spark.sql.adaptive.enabled", "true")
    .config("spark.sql.adaptive.coalescePartitions.enabled", "true")
    .config("spark.sql.adaptive.skewJoin.enabled", "true")
    
    # Delta Lake optimizations
    .config("spark.sql.extensions", "io.delta.sql.DeltaSparkSessionExtension")
    .config("spark.sql.catalog.spark_catalog", "org.apache.spark.sql.delta.catalog.DeltaCatalog")
    
    .getOrCreate()
)
```

### Handling Data Skew

Data skew is the #1 Spark performance problem in production.

```python
# Symptom: One executor takes 10x longer than others
# Cause: Some keys have far more data (e.g., 30% of orders are from 1 customer)

# Solution 1: Salting (add random prefix to key, process, then reduce)
from pyspark.sql import functions as F
import random

SALT_BUCKETS = 20

df_salted = large_df.withColumn(
    "salted_key",
    F.concat(F.lit(str(random.randint(0, SALT_BUCKETS))), F.lit("_"), F.col("customer_id"))
)

# Join with the small table, which also needs salting
small_df_exploded = small_df.withColumn(
    "salt", F.array([F.lit(str(i)) for i in range(SALT_BUCKETS)])
).withColumn("salt", F.explode("salt")) \
 .withColumn("salted_key", F.concat(F.col("salt"), F.lit("_"), F.col("customer_id")))

result = df_salted.join(small_df_exploded, "salted_key")

# Solution 2: Spark 3 AQE Skew Join (automatic with AQE enabled above)
# AQE detects skewed partitions and splits them automatically
```

### Write Optimization

```python
# Always write Parquet, not CSV
# Always partition by date for time-series data
# Use Z-ORDER on high-cardinality filter columns (Delta Lake)

df.write \
    .format("delta") \
    .mode("overwrite") \
    .partitionBy("order_date") \
    .option("overwriteSchema", "true") \
    .save("s3://bucket/delta/orders/")

# After bulk writes, run OPTIMIZE + ZORDER
spark.sql("""
    OPTIMIZE delta.`s3://bucket/delta/orders/`
    ZORDER BY (customer_id, product_id)
""")
```

---

## 11. Cloud Architecture Best Practices

### Multi-Region vs Single-Region

```
Single-region:
  ✓ Simpler, cheaper
  ✗ Single datacenter failure = total outage
  → Use for: non-critical pipelines, dev/staging

Multi-region active-passive:
  ✓ Disaster recovery (RPO < 1 hour, RTO < 4 hours)
  ✗ More complex, ~2x cost
  → Use for: critical business data, financial data

Multi-region active-active:
  ✓ Zero downtime, data-residency compliance
  ✗ Complex conflict resolution, ~3x cost
  → Use for: global user data, real-time globally
```

### Environment Separation

```
dev:     Developer laptops + cloud sandbox
         → Small datasets, sample data only
         → No real user data allowed
         → Cheap: pay-per-query (Athena, BigQuery)

staging: Full-scale, anonymized copy of prod data
         → Test with production-like volumes
         → Run full integration tests
         → Replicate infrastructure to catch Terraform bugs

prod:    Real data, real users
         → Change freeze windows
         → Canary deployments
         → Rollback plan for every change
```

### The Medallion Architecture in Practice

```
Bronze (raw):    Land data exactly as received — never modify bronze
                 Schema: {source_fields...} + _ingested_at, _source_file
                 Retention: 90 days (compliance requirement)
                 
Silver (clean):  Validated, deduplicated, standardized
                 Schema: same as bronze + _is_valid, _cleaned_at
                 Transformations: type casting, null handling, deduplication
                 No business logic here
                 
Gold (business): Business-ready aggregates and dimensional models
                 Fact tables, dimension tables, aggregates
                 Business logic applied (revenue = price × quantity × (1-discount))
                 These are the tables the business reads from
```

---

## 12. Team & Process Best Practices

### Documentation as Code

```yaml
# dbt schema.yml — documentation lives with the model
version: 2

models:
  - name: fct_orders
    description: |
      One row per order. Revenue is in USD, converted from source currency
      using daily exchange rates from the fx_rates table. Cancelled orders
      are excluded. Late-arriving orders (up to 3 days) are handled by the
      lookback window.
    
    meta:
      owner: "@data-team"
      sla: "Available by 8 AM UTC daily"
    
    columns:
      - name: order_id
        description: Unique identifier for the order (from Stripe)
        tests:
          - unique
          - not_null
      - name: revenue
        description: |
          Order revenue in USD. This is the amount charged to the customer
          after discounts, before refunds. Does NOT include shipping costs.
        tests:
          - not_null
          - dbt_utils.expression_is_true:
              expression: ">= 0"
```

### Code Review Standards for Data

**What to check in a data PR:**
- [ ] Is the transformation logic correct? (test with sample data)
- [ ] Are tests added for new columns?
- [ ] Is the model description updated?
- [ ] Will this handle late-arriving data?
- [ ] Is it idempotent?
- [ ] What's the performance impact? (check EXPLAIN plan)
- [ ] Does it handle NULLs correctly?
- [ ] Is PII handled appropriately?

### On-Call Runbooks

Every critical pipeline should have a runbook:

```markdown
# Daily Orders Pipeline — On-Call Runbook

## Alert: daily_orders_pipeline DAG failed

### Step 1: Check the failure
1. Open Airflow UI → DAGs → daily_orders_pipeline
2. Click on the failed run → Click failed task → View logs
3. Common failures:
   - `Connection refused (S3)` → Check AWS credentials rotation
   - `Table not found` → Check if upstream schema changed (run `dbt compile`)
   - `OOM error` → Reduce batch size (change CHUNK_SIZE=50000 in config)

### Step 2: Re-run safely
1. Verify the pipeline is idempotent (it is — uses DELETE+INSERT)
2. Clear the failed task in Airflow
3. Trigger the failed run for the specific date

### Step 3: Escalate if...
- Data has been missing for > 4 hours
- Re-run fails 3+ times in a row
- You see schema mismatch errors (upstream contract broken)
→ Page the team lead
```

---

## 13. The 81 Platform Design Questions

Before building a data platform, answer these questions. They come from real engineering decisions made by senior data engineers.

### Data Source Questions (What are you ingesting?)

**Volume & Velocity:**
1. How much data per day? (MB → GB → TB → PB determines tool choice)
2. What's the peak event rate? (events/second)
3. What's the acceptable latency? (real-time, near-real-time, daily batch?)
4. How long do you need to retain raw data? (7 days? 7 years?)

**Source characteristics:**
5. Is the source a database, file system, API, or message queue?
6. Does the source support CDC/streaming, or only bulk export?
7. What format is the source data? (JSON, CSV, Avro, Parquet, binary)
8. Is there a schema? Is it enforced? Can it change?
9. How reliable is the source? (can it go down during your extract window?)
10. Is there a rate limit on the API?
11. What time zone are timestamps in? Are they consistent?
12. Are there soft deletes or hard deletes in the source?
13. How do you handle late-arriving data from this source?

**Business Questions:**
14. What is this data used for? (reporting, ML, operations?)
15. Who are the data consumers? (business analysts, data scientists, applications)
16. What's the SLA for data availability?
17. What's the cost of data being wrong?
18. What's the cost of data being late?
19. Is there PII in this data? What are the compliance requirements?

### Goals and Destination Questions (What are you building?)

**Output:**
20. What is the final data product? (dashboard, API, ML feature, report)
21. What tool will consumers use to access the data? (SQL, Python, BI tool)
22. What query patterns will consumers run? (aggregations, point lookups, time-series)
23. How many concurrent users will query the data?
24. What's the acceptable query latency? (< 1 second, < 1 minute, hours)

**Architecture:**
25. Batch or streaming? What's the minimum acceptable latency?
26. Lambda (batch + streaming) or Kappa (streaming only)?
27. Which cloud provider? Multi-cloud? On-prem?
28. What's your budget? (this eliminates many options immediately)
29. What's your team's current skill set? (favor tools your team knows)

**Operations:**
30. Who owns this pipeline long-term? (dedicated DE team or shared ownership)
31. What's the DR/HA requirement? (can the pipeline be down for 4 hours?)
32. How do you monitor pipeline health?
33. What's the on-call expectation?
34. How do you handle schema evolution?
35. How do you roll back a bad deploy?

*The full 81 questions continue across pipeline design, testing, security, compliance, and production readiness — answering these upfront saves weeks of rework.*

---

## Summary: The Best Practices Cheat Sheet

```
Pipeline Design
├── Make every pipeline idempotent (re-runnable = same result)
├── Handle late data with lookback windows or watermarks
├── Design for backfill from day one (parameterize by date)
└── Validate before loading (row counts, nulls, ranges)

Performance
├── Partition tables by date (99% of queries filter by time)
├── Broadcast small tables in Spark joins
├── Use Parquet/ORC — never CSV in production
└── Profile before optimizing — measure, don't guess

Security
├── Least privilege — applications don't need admin
├── Secrets in secret manager — never in code or env files
├── Encrypt PII — hash emails before landing in data lake
└── GDPR: design deletion from day one — retrofitting is expensive

Reliability
├── Test at every layer: unit, integration, contract
├── Monitor freshness, row counts, and schema changes
└── Write runbooks for on-call — future you is on-call too

Collaboration
├── Document your data with dbt schema.yml
├── Define data contracts with producers before ingestion
└── Use CI/CD — data changes deserve the same rigor as code
```

---

← [11 Case Studies](11-case-studies.md) | [13 Interview Prep →](13-interview-prep.md)
