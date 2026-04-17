# 09 — Hands-On Data Engineering Course

> **Guide by [Mano Harsha Sappa](https://www.linkedin.com/in/manoharshasappa/)** | [Portfolio](https://manoharshasappa.github.io/portfolio_ManoHarshaSappa/) | [Back to Index](../README.md)

> Based on the **Data Engineering Zoomcamp** by DataTalks.Club — a free 9-week course. Supplemented with additional hands-on content.

---

## Course Overview

This 9-week course takes you from zero to a complete end-to-end data engineering pipeline. You will build real systems with production-grade tools.

**Prerequisites**:
- Basic coding experience (Python helpful)
- SQL familiarity
- No prior data engineering experience needed

**Tools Used**:
- Docker + Docker Compose
- Terraform
- Kestra (Workflow Orchestration)
- Google BigQuery (Data Warehouse)
- dbt (Analytics Engineering)
- Apache Spark (Batch Processing)
- Apache Kafka (Stream Processing)
- Avro + Schema Registry

**Final Goal**: Build a complete end-to-end data pipeline from data ingestion → processing → warehouse → dashboard.

---

## Module 1: Containerization & Infrastructure as Code

### Learning Objectives
- Run PostgreSQL and Python in Docker containers
- Set up a local data environment with Docker Compose
- Provision cloud infrastructure with Terraform
- Understand GCP basics

### 1.1 Docker + PostgreSQL Setup

```bash
# Pull and run PostgreSQL
docker run -d \
    --name postgres-dev \
    -e POSTGRES_USER=admin \
    -e POSTGRES_PASSWORD=secret \
    -e POSTGRES_DB=ny_taxi \
    -p 5432:5432 \
    -v pgdata:/var/lib/postgresql/data \
    postgres:15

# Verify it's running
docker ps
docker logs postgres-dev
```

### 1.2 Ingesting Data into PostgreSQL

```python
#!/usr/bin/env python
# ingest_data.py — Ingest NY Taxi CSV into PostgreSQL

import pandas as pd
from sqlalchemy import create_engine
import argparse
import sys

def ingest_data(user, password, host, port, db, table_name, url):
    # Download the data
    print(f"Downloading data from {url}")
    df_iter = pd.read_csv(url, iterator=True, chunksize=100000)

    engine = create_engine(f'postgresql://{user}:{password}@{host}:{port}/{db}')

    # Create table from first chunk
    df = next(df_iter)
    df['tpep_pickup_datetime'] = pd.to_datetime(df['tpep_pickup_datetime'])
    df['tpep_dropoff_datetime'] = pd.to_datetime(df['tpep_dropoff_datetime'])

    df.head(0).to_sql(name=table_name, con=engine, if_exists='replace')
    df.to_sql(name=table_name, con=engine, if_exists='append')
    print(f"Inserted first {len(df):,} rows")

    # Process remaining chunks
    chunk_num = 1
    while True:
        try:
            df_chunk = next(df_iter)
            df_chunk['tpep_pickup_datetime'] = pd.to_datetime(df_chunk['tpep_pickup_datetime'])
            df_chunk['tpep_dropoff_datetime'] = pd.to_datetime(df_chunk['tpep_dropoff_datetime'])
            df_chunk.to_sql(name=table_name, con=engine, if_exists='append')
            chunk_num += 1
            print(f"Inserted chunk {chunk_num}: {len(df_chunk):,} rows")
        except StopIteration:
            print("Finished loading all data!")
            break

if __name__ == '__main__':
    parser = argparse.ArgumentParser(description='Ingest CSV data to PostgreSQL')
    parser.add_argument('--user', help='PostgreSQL user')
    parser.add_argument('--password', help='PostgreSQL password')
    parser.add_argument('--host', help='PostgreSQL host')
    parser.add_argument('--port', help='PostgreSQL port', default='5432')
    parser.add_argument('--db', help='Database name')
    parser.add_argument('--table_name', help='Target table name')
    parser.add_argument('--url', help='CSV URL')
    args = parser.parse_args()

    ingest_data(args.user, args.password, args.host, args.port, args.db, args.table_name, args.url)
```

### 1.3 Docker Compose — Full Local Stack

```yaml
# docker-compose.yml
version: '3.8'

services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: ny_taxi
    volumes:
      - pgdata:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  pgadmin:
    image: dpage/pgadmin4:latest
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@example.com
      PGADMIN_DEFAULT_PASSWORD: secret
    ports:
      - "8080:80"
    depends_on:
      - postgres

  ingest:
    build: .
    command: >
      python ingest_data.py
        --user=admin
        --password=secret
        --host=postgres
        --port=5432
        --db=ny_taxi
        --table_name=yellow_taxi_trips
        --url=https://github.com/DataTalksClub/nyc-tlc-data/releases/download/yellow/yellow_tripdata_2021-01.csv.gz
    depends_on:
      - postgres

volumes:
  pgdata:
```

### 1.4 Terraform — Infrastructure as Code

```hcl
# main.tf — Provision GCP BigQuery + GCS

terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
}

provider "google" {
  project = var.project_id
  region  = var.region
}

variable "project_id" {}
variable "region" { default = "us-central1" }

# GCS Data Lake Bucket
resource "google_storage_bucket" "data_lake" {
  name          = "${var.project_id}-datalake"
  location      = "US"
  force_destroy = true

  lifecycle_rule {
    condition { age = 30 }
    action { type = "Delete" }
  }
}

# BigQuery Dataset
resource "google_bigquery_dataset" "raw" {
  dataset_id = "raw"
  location   = "US"
  delete_contents_on_destroy = true
}

resource "google_bigquery_dataset" "prod" {
  dataset_id = "production"
  location   = "US"
}

# BigQuery Table with partitioning
resource "google_bigquery_table" "ny_taxi" {
  dataset_id = google_bigquery_dataset.raw.dataset_id
  table_id   = "ny_taxi_trips"
  deletion_protection = false

  time_partitioning {
    type  = "DAY"
    field = "tpep_pickup_datetime"
  }

  clustering = ["vendor_id", "payment_type"]

  schema = jsonencode([
    {name="vendor_id", type="STRING"},
    {name="tpep_pickup_datetime", type="TIMESTAMP"},
    {name="tpep_dropoff_datetime", type="TIMESTAMP"},
    {name="passenger_count", type="FLOAT64"},
    {name="trip_distance", type="FLOAT64"},
    {name="fare_amount", type="FLOAT64"},
    {name="total_amount", type="FLOAT64"}
  ])
}

output "bucket_name" {
  value = google_storage_bucket.data_lake.name
}
```

```bash
# Terraform workflow
gcloud auth application-default login
terraform init
terraform plan
terraform apply
terraform destroy    # Clean up resources when done
```

### Module 1 Homework

1. Run a PostgreSQL database with Docker
2. Ingest the NY Taxi January 2021 dataset (~1.3M rows)
3. Answer SQL queries:
   - How many trips had fare > $100?
   - What percentage of trips had 0 passengers?
   - Which vendor had the highest average tip?
4. Provision BigQuery + GCS with Terraform
5. Load the same NY Taxi data to BigQuery

---

## Module 2: Workflow Orchestration with Kestra

### Learning Objectives
- Understand workflow orchestration concepts (DAGs, tasks, triggers)
- Build data ingestion workflows in Kestra
- Handle scheduling, retries, and error notifications

### 2.1 Kestra Setup

```bash
# Start Kestra with Docker Compose
docker run --pull=always \
    --rm -it \
    -p 8080:8080 \
    --user=root \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -v /tmp:/tmp \
    kestra/kestra:latest server local
```

Access Kestra UI at http://localhost:8080

### 2.2 Your First Kestra Flow

```yaml
# ny_taxi_ingest.yaml
id: ny-taxi-ingest
namespace: data.engineering

description: "Ingest NY Taxi data to GCS and BigQuery"

inputs:
  - name: year
    type: INT
    defaults: 2021
  - name: month
    type: INT
    defaults: 1

triggers:
  - id: monthly-schedule
    type: io.kestra.core.models.triggers.types.Schedule
    cron: "0 4 1 * *"  # 4am on the 1st of each month

variables:
  file_name: "yellow_tripdata_{{inputs.year}}-{{inputs.month | numberFormat('00')}}.parquet"
  gcs_path: "gs://my-bucket/raw/ny_taxi/{{vars.file_name}}"

tasks:
  - id: download-data
    type: io.kestra.plugin.fs.http.Download
    uri: "https://d37ci6vzurychx.cloudfront.net/trip-data/{{vars.file_name}}"
    description: "Download NY Taxi data for {{inputs.year}}-{{inputs.month}}"

  - id: upload-to-gcs
    type: io.kestra.plugin.gcp.gcs.Upload
    from: "{{outputs['download-data'].uri}}"
    to: "{{vars.gcs_path}}"
    serviceAccount: "{{secret('GCP_SERVICE_ACCOUNT')}}"
    projectId: "{{secret('GCP_PROJECT_ID')}}"

  - id: create-bq-table
    type: io.kestra.plugin.gcp.bigquery.CreateTable
    projectId: "{{secret('GCP_PROJECT_ID')}}"
    dataset: raw
    table: "ny_taxi_{{inputs.year}}_{{inputs.month | numberFormat('00')}}"
    schema:
      - name: vendor_id
        type: STRING
      - name: tpep_pickup_datetime
        type: TIMESTAMP
      - name: fare_amount
        type: FLOAT64
    timePartitioningField: tpep_pickup_datetime

  - id: load-to-bigquery
    type: io.kestra.plugin.gcp.bigquery.Load
    from: "{{vars.gcs_path}}"
    projectId: "{{secret('GCP_PROJECT_ID')}}"
    dataset: raw
    table: "ny_taxi_{{inputs.year}}_{{inputs.month | numberFormat('00')}}"
    format: PARQUET
    writeDisposition: WRITE_TRUNCATE

  - id: notify-success
    type: io.kestra.plugin.notifications.slack.SlackIncomingWebhook
    url: "{{secret('SLACK_WEBHOOK')}}"
    payload: |
      {"text": "✅ NY Taxi data loaded for {{inputs.year}}-{{inputs.month}}. File: {{vars.file_name}}"}

errors:
  - id: notify-failure
    type: io.kestra.plugin.notifications.slack.SlackIncomingWebhook
    url: "{{secret('SLACK_WEBHOOK')}}"
    payload: |
      {"text": "❌ NY Taxi pipeline FAILED for {{inputs.year}}-{{inputs.month}}. Error: {{errorLogs()}}"}
```

---

## Module 3: Data Warehousing with BigQuery

### Learning Objectives
- Understand BigQuery internals (Capacitor columnar format, Dremel execution)
- Partitioning and clustering for performance and cost
- BigQuery best practices
- Machine Learning in BigQuery

### 3.1 Partitioning in BigQuery

```sql
-- Create external table from GCS
CREATE OR REPLACE EXTERNAL TABLE `project.raw.ny_taxi_external`
OPTIONS (
    format = 'PARQUET',
    uris = ['gs://my-bucket/raw/ny_taxi/*.parquet']
);

-- Create native partitioned table (much faster)
CREATE OR REPLACE TABLE `project.raw.ny_taxi`
PARTITION BY DATE(tpep_pickup_datetime)
CLUSTER BY vendor_id, payment_type
AS
SELECT * FROM `project.raw.ny_taxi_external`;

-- Query with partition pruning (only reads 1 day's data)
SELECT COUNT(*), AVG(fare_amount), AVG(trip_distance)
FROM `project.raw.ny_taxi`
WHERE DATE(tpep_pickup_datetime) = '2021-01-15'  -- Partition pruned!
    AND vendor_id = '1';

-- Check partition info
SELECT table_name, partition_id, row_count, total_logical_bytes
FROM `project.raw.INFORMATION_SCHEMA.PARTITIONS`
WHERE table_name = 'ny_taxi'
ORDER BY partition_id;
```

### 3.2 BigQuery ML

```sql
-- Train a tip prediction model
CREATE OR REPLACE MODEL `project.ml.tip_model`
OPTIONS(
    model_type='linear_reg',
    input_label_cols=['tip_amount']
) AS
SELECT
    vendor_id,
    passenger_count,
    trip_distance,
    payment_type,
    fare_amount,
    hour_of_day,
    day_of_week,
    tip_amount
FROM (
    SELECT *,
        EXTRACT(HOUR FROM tpep_pickup_datetime) AS hour_of_day,
        EXTRACT(DAYOFWEEK FROM tpep_pickup_datetime) AS day_of_week
    FROM `project.raw.ny_taxi`
    WHERE tip_amount IS NOT NULL AND tip_amount >= 0
);

-- Evaluate model
SELECT * FROM ML.EVALUATE(MODEL `project.ml.tip_model`);

-- Make predictions
SELECT *
FROM ML.PREDICT(MODEL `project.ml.tip_model`,
    (SELECT vendor_id, passenger_count, trip_distance, payment_type,
        fare_amount, EXTRACT(HOUR FROM CURRENT_TIMESTAMP) AS hour_of_day,
        EXTRACT(DAYOFWEEK FROM CURRENT_TIMESTAMP) AS day_of_week
     FROM prediction_data)
);
```

---

## Module 4: Analytics Engineering with dbt

### Learning Objectives
- Set up dbt with BigQuery
- Build staging, intermediate, and mart models
- Write tests and documentation
- Deploy dbt in production (CI/CD)

### 4.1 dbt Project Setup

```bash
# Install dbt with BigQuery adapter
pip install dbt-bigquery

# Initialize project
dbt init ny_taxi_analytics
cd ny_taxi_analytics

# Configure BigQuery connection in profiles.yml
cat > ~/.dbt/profiles.yml << 'EOF'
ny_taxi_analytics:
  target: dev
  outputs:
    dev:
      type: bigquery
      method: oauth
      project: my-project-id
      dataset: dbt_dev
      threads: 4
      timeout_seconds: 300
    prod:
      type: bigquery
      method: service-account
      project: my-project-id
      dataset: production
      keyfile: /path/to/service_account.json
      threads: 8
EOF

# Test connection
dbt debug
```

### 4.2 NY Taxi dbt Models

```sql
-- models/staging/stg_ny_taxi__trips.sql
WITH source AS (
    SELECT * FROM {{ source('raw', 'ny_taxi') }}
),

staged AS (
    SELECT
        -- Identifiers
        {{ dbt_utils.generate_surrogate_key(['vendor_id', 'tpep_pickup_datetime', 'tpep_dropoff_datetime']) }} AS trip_id,
        vendor_id,
        rate_code_id,
        payment_type_id AS payment_type,
        pu_location_id AS pickup_location_id,
        do_location_id AS dropoff_location_id,

        -- Timestamps
        tpep_pickup_datetime AS pickup_datetime,
        tpep_dropoff_datetime AS dropoff_datetime,
        DATE(tpep_pickup_datetime) AS pickup_date,

        -- Metrics
        passenger_count,
        trip_distance,
        fare_amount,
        extra,
        mta_tax,
        tip_amount,
        tolls_amount,
        improvement_surcharge,
        total_amount,

        -- Derived
        TIMESTAMP_DIFF(tpep_dropoff_datetime, tpep_pickup_datetime, MINUTE) AS trip_duration_minutes,
        CASE
            WHEN trip_distance > 0 THEN fare_amount / trip_distance
            ELSE NULL
        END AS fare_per_mile

    FROM source
    WHERE vendor_id IS NOT NULL
        AND tpep_pickup_datetime IS NOT NULL
        AND tpep_dropoff_datetime > tpep_pickup_datetime
        AND trip_distance >= 0
        AND fare_amount >= 0
)

SELECT * FROM staged
```

```sql
-- models/marts/fct_daily_trips.sql
{{
    config(
        materialized='incremental',
        unique_key='pickup_date',
        partition_by={'field': 'pickup_date', 'data_type': 'date'}
    )
}}

SELECT
    pickup_date,
    vendor_id,
    payment_type,
    COUNT(*) AS trip_count,
    SUM(trip_distance) AS total_distance,
    AVG(trip_distance) AS avg_distance,
    SUM(fare_amount) AS total_fare,
    AVG(fare_amount) AS avg_fare,
    SUM(tip_amount) AS total_tips,
    AVG(tip_amount) AS avg_tip,
    AVG(trip_duration_minutes) AS avg_duration_minutes,
    SUM(total_amount) AS total_revenue,
    COUNT(DISTINCT pickup_location_id) AS unique_pickup_zones
FROM {{ ref('stg_ny_taxi__trips') }}

{% if is_incremental() %}
    WHERE pickup_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 3 DAY)
{% endif %}

GROUP BY 1, 2, 3
```

### 4.3 Run dbt

```bash
dbt run                          # Run all models
dbt test                         # Run all tests
dbt run --select stg_ny_taxi+    # Run staging and all downstream

# Production deployment with CI/CD
dbt run --target prod --select state:modified+
dbt test --target prod
dbt docs generate
```

---

## Module 5: Data Platforms — End-to-End with Bruin

### Learning Objectives
- Build complete pipelines with Bruin (open-source data platform)
- Data ingestion, transformation, and quality in one tool
- Deploy to cloud (BigQuery)

```yaml
# pipeline.yml — Bruin pipeline definition
name: ny_taxi_pipeline
schedule: "@daily"

connections:
  - name: gcp-connection
    type: google-cloud-platform
    project: my-project
    location: US

assets:
  - name: raw.ny_taxi
    type: bq.sql
    description: "Raw NY Taxi data loaded from GCS"
    parameters:
      table: raw.ny_taxi
    upstream: []

  - name: staging.stg_trips
    type: bq.sql
    description: "Cleaned NY Taxi trips"
    upstream:
      - raw.ny_taxi
    custom_checks:
      - name: row_count
        query: "SELECT COUNT(*) FROM staging.stg_trips WHERE pickup_date = DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY)"
        value: "> 10000"
```

---

## Module 6: Batch Processing with Apache Spark

### Learning Objectives
- Set up Spark locally with Docker
- Process large datasets with PySpark DataFrames
- Understand Spark internals (partitions, shuffles, memory)
- Submit Spark jobs to a cluster

### 6.1 Local Spark Setup

```bash
# Run Spark with Jupyter
docker run -it --rm \
    -p 8888:8888 \
    -p 4040:4040 \
    -v $(pwd):/home/jovyan/work \
    jupyter/pyspark-notebook:latest
```

### 6.2 Processing NY Taxi Data with Spark

```python
from pyspark.sql import SparkSession
from pyspark.sql import functions as F
from pyspark.sql.types import *

# Initialize Spark
spark = SparkSession.builder \
    .appName("NY Taxi Analysis") \
    .config("spark.sql.shuffle.partitions", "8") \
    .config("spark.driver.memory", "4g") \
    .getOrCreate()

# Read parquet files
df = spark.read.parquet("data/ny_taxi/*.parquet")

print(f"Total records: {df.count():,}")
print(f"Partitions: {df.rdd.getNumPartitions()}")
df.printSchema()

# Repartition for better parallelism
df = df.repartition(8)

# Filter and transform
trips = df \
    .filter(F.col("tpep_pickup_datetime").isNotNull()) \
    .filter(F.col("fare_amount") >= 0) \
    .filter(F.col("trip_distance") >= 0) \
    .withColumn("pickup_date", F.to_date(F.col("tpep_pickup_datetime"))) \
    .withColumn("pickup_hour", F.hour(F.col("tpep_pickup_datetime"))) \
    .withColumn(
        "trip_duration_min",
        (F.col("tpep_dropoff_datetime").cast("long") -
         F.col("tpep_pickup_datetime").cast("long")) / 60
    ) \
    .filter(F.col("trip_duration_min").between(1, 1440))  # 1 min to 24 hours

# Daily aggregation
daily = trips \
    .groupBy("pickup_date", "vendor_id") \
    .agg(
        F.count("*").alias("trip_count"),
        F.sum("fare_amount").alias("total_fare"),
        F.avg("fare_amount").alias("avg_fare"),
        F.sum("tip_amount").alias("total_tips"),
        F.avg("trip_distance").alias("avg_distance"),
        F.avg("trip_duration_min").alias("avg_duration")
    ) \
    .orderBy("pickup_date")

daily.show(10)

# Revenue by hour of day
hourly = trips \
    .groupBy("pickup_hour") \
    .agg(
        F.count("*").alias("trip_count"),
        F.avg("fare_amount").alias("avg_fare")
    ) \
    .orderBy("pickup_hour")

hourly.show(24)

# Write results to GCS (or local)
daily.write \
    .partitionBy("pickup_date") \
    .parquet("gs://my-bucket/processed/ny_taxi_daily/", mode="overwrite")

# Use SparkSQL
trips.createOrReplaceTempView("ny_taxi_trips")

top_zones = spark.sql("""
    SELECT
        pu_location_id AS zone,
        COUNT(*) AS trips,
        ROUND(AVG(fare_amount), 2) AS avg_fare,
        ROUND(SUM(fare_amount), 2) AS total_revenue
    FROM ny_taxi_trips
    WHERE pickup_date >= '2021-01-01'
    GROUP BY 1
    ORDER BY total_revenue DESC
    LIMIT 20
""")

top_zones.show()
```

### 6.3 Spark GroupBy Internals

```python
# Understanding shuffles
# GroupBy requires shuffle (expensive — data moves across network)

# Check the physical plan
df.groupBy("vendor_id").count().explain(extended=True)

# Optimization: Use reduceByKey (local pre-aggregation) instead of groupByKey
# (RDD-level optimization — modern code uses DataFrames which optimize this automatically)

# Broadcast join: for small tables (< few hundred MB)
zones_df = spark.read.csv("taxi_zones.csv", header=True, inferSchema=True)
trips_with_zones = trips.join(F.broadcast(zones_df),
    trips.pu_location_id == zones_df.location_id,
    "left"
)
```

---

## Module 7: Stream Processing with Kafka

### Learning Objectives
- Set up Kafka with Docker
- Produce and consume events in Python
- Schema management with Avro + Schema Registry
- Process streams with Kafka Streams and KSQL

### 7.1 Kafka + Schema Registry Setup

```yaml
# docker-compose-kafka.yml
version: '3.8'

services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.4.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181

  kafka:
    image: confluentinc/cp-kafka:7.4.0
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: 'true'

  schema-registry:
    image: confluentinc/cp-schema-registry:7.4.0
    depends_on:
      - kafka
    ports:
      - "8081:8081"
    environment:
      SCHEMA_REGISTRY_HOST_NAME: schema-registry
      SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: kafka:9092

  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    ports:
      - "8080:8080"
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:9092
      KAFKA_CLUSTERS_0_SCHEMAREGISTRY: http://schema-registry:8081
```

### 7.2 Avro Schema + Producer

```python
# ride_schema.py
RIDE_KEY_SCHEMA = """
{
    "type": "record",
    "name": "RideKey",
    "fields": [
        {"name": "vendor_id", "type": "string"},
        {"name": "pickup_datetime", "type": {"type": "long", "logicalType": "timestamp-millis"}}
    ]
}
"""

RIDE_VALUE_SCHEMA = """
{
    "type": "record",
    "name": "Ride",
    "fields": [
        {"name": "vendor_id", "type": "string"},
        {"name": "passenger_count", "type": ["null", "int"], "default": null},
        {"name": "trip_distance", "type": "double"},
        {"name": "pickup_location_id", "type": "int"},
        {"name": "dropoff_location_id", "type": "int"},
        {"name": "fare_amount", "type": "double"},
        {"name": "total_amount", "type": "double"},
        {"name": "pickup_datetime", "type": {"type": "long", "logicalType": "timestamp-millis"}}
    ]
}
"""
```

```python
# producer.py
from confluent_kafka import Producer
from confluent_kafka.schema_registry import SchemaRegistryClient
from confluent_kafka.schema_registry.avro import AvroSerializer
from confluent_kafka.serialization import SerializationContext, MessageField
import pandas as pd
from datetime import datetime
import time

schema_registry_client = SchemaRegistryClient({'url': 'http://localhost:8081'})

avro_serializer = AvroSerializer(
    schema_registry_client=schema_registry_client,
    schema_str=RIDE_VALUE_SCHEMA
)

producer = Producer({'bootstrap.servers': 'localhost:9092'})

def send_ride(ride: dict):
    """Send a ride event to Kafka."""
    producer.produce(
        topic='rides',
        key=str(ride['vendor_id']),
        value=avro_serializer(ride, SerializationContext('rides', MessageField.VALUE)),
        on_delivery=delivery_report
    )

def delivery_report(err, msg):
    if err:
        print(f'Message delivery failed: {err}')

# Simulate streaming from CSV
df = pd.read_parquet('yellow_tripdata_2021-01.parquet')
for _, row in df.iterrows():
    ride = {
        'vendor_id': str(row['VendorID']),
        'passenger_count': int(row['passenger_count']) if pd.notna(row['passenger_count']) else None,
        'trip_distance': float(row['trip_distance']),
        'pickup_location_id': int(row['PULocationID']),
        'dropoff_location_id': int(row['DOLocationID']),
        'fare_amount': float(row['fare_amount']),
        'total_amount': float(row['total_amount']),
        'pickup_datetime': int(row['tpep_pickup_datetime'].timestamp() * 1000)
    }
    send_ride(ride)
    time.sleep(0.01)  # Simulate 100 events/second

producer.flush()
print("Done!")
```

### 7.3 Stream Processing with Spark Structured Streaming

```python
from pyspark.sql import SparkSession
from pyspark.sql import functions as F
from pyspark.sql.avro.functions import from_avro

spark = SparkSession.builder \
    .appName("Ride Stream Processor") \
    .config("spark.jars.packages",
        "org.apache.spark:spark-sql-kafka-0-10_2.12:3.4.0,"
        "org.apache.spark:spark-avro_2.12:3.4.0") \
    .getOrCreate()

# Read from Kafka
raw_stream = spark.readStream \
    .format("kafka") \
    .option("kafka.bootstrap.servers", "localhost:9092") \
    .option("subscribe", "rides") \
    .option("startingOffsets", "earliest") \
    .load()

# Parse Avro with schema
rides = raw_stream \
    .select(from_avro(F.col("value"), RIDE_VALUE_SCHEMA).alias("ride")) \
    .select("ride.*") \
    .withColumn("pickup_time", F.to_timestamp(F.col("pickup_datetime") / 1000))

# Windowed aggregation (5-minute windows, updated every 30 seconds)
windowed_counts = rides \
    .withWatermark("pickup_time", "10 minutes") \
    .groupBy(
        F.window(F.col("pickup_time"), "5 minutes", "30 seconds"),
        F.col("vendor_id")
    ) \
    .agg(
        F.count("*").alias("trip_count"),
        F.sum("fare_amount").alias("total_fare"),
        F.avg("trip_distance").alias("avg_distance")
    )

# Write to console for testing
query = windowed_counts.writeStream \
    .outputMode("append") \
    .format("console") \
    .option("truncate", False) \
    .start()

query.awaitTermination()
```

---

## Final Project Guide

### Project Requirements

Build a complete end-to-end pipeline that:

1. **Ingests** data from at least one real data source
2. **Processes** the data in batch or streaming
3. **Stores** in a data warehouse (BigQuery, Snowflake, or Redshift)
4. **Transforms** with dbt models (staging + mart layer)
5. **Visualizes** with a dashboard (Google Looker Studio, Metabase, or Tableau)
6. **Orchestrates** with Airflow, Prefect, or Kestra
7. **Documents** the pipeline (architecture diagram + README)

### Suggested Datasets

| Dataset | Source | Format |
|---------|--------|--------|
| NY Yellow Taxi Trips | DataTalks.Club GitHub | Parquet |
| NYC Green Taxi | DataTalks.Club GitHub | Parquet |
| NOAA Climate Data | NOAA API | JSON/CSV |
| GitHub Archive | githubarchive.org | JSON |
| Divvy Bikeshare | Divvy Bikes | CSV |
| OpenAQ Air Quality | openaq.org | JSON API |
| Spotify Charts | Kaggle | CSV |
| COVID-19 Data | Our World in Data | CSV |
| US Stock Market | Yahoo Finance API | JSON |

### Evaluation Criteria

- Pipeline runs end-to-end without manual intervention
- Data is partitioned appropriately in the warehouse
- dbt models have at least 5 tests
- Dashboard shows 3+ meaningful visualizations
- README explains the architecture

---

*← [08 - dbt Modeling](08-dbt-modeling.md) | [10 - Tools Catalog](10-tools-catalog.md) →*
