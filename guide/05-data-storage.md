# 05 — Data Storage Systems

> **Guide by [Mano Harsha Sappa](https://www.linkedin.com/in/manoharshasappa/)** | [Portfolio](https://manoharshasappa.github.io/portfolio_ManoHarshaSappa/) | [Back to Index](../README.md)

---

## Contents

- [SQL Databases](#sql-databases)
- [NoSQL Databases](#nosql-databases)
- [Data Warehouses](#data-warehouses)
- [Data Lakes](#data-lakes)
- [Data Lakehouses](#data-lakehouses-open-table-formats)
- [Serialization Formats](#serialization-formats)
- [Choosing the Right Storage](#choosing-the-right-storage)

---

## SQL Databases

Relational databases are the foundation of most production systems. As a data engineer, you'll regularly ingest from and write to SQL databases.

### PostgreSQL — The Enterprise Open-Source Standard

PostgreSQL is the most advanced open-source RDBMS. It's the default choice for production applications.

```sql
-- Create table with proper types
CREATE TABLE users (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email       VARCHAR(255) UNIQUE NOT NULL,
    name        VARCHAR(255),
    created_at  TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at  TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    metadata    JSONB                                      -- Native JSON support!
);

-- Create index for frequent queries
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_created_at ON users(created_at DESC);
CREATE INDEX idx_metadata ON users USING GIN(metadata);   -- JSON index

-- Stored procedure / function
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER set_updated_at
    BEFORE UPDATE ON users
    FOR EACH ROW EXECUTE FUNCTION update_updated_at();
```

### Database Design Principles

**ACID Properties** (what makes relational DBs reliable):
- **Atomicity**: A transaction either fully succeeds or fully fails — no partial updates
- **Consistency**: Data always moves from one valid state to another
- **Isolation**: Concurrent transactions don't interfere with each other
- **Durability**: Committed transactions persist even after system crashes

**Normalization**: Organizing tables to reduce redundancy
- **1NF**: Each column holds atomic values (no lists, no repeating groups)
- **2NF**: No partial dependencies on composite primary key
- **3NF**: No transitive dependencies (A → B → C should be A → C directly)

**When NOT to normalize**: For analytics/OLAP, denormalized (wide) tables are often faster because joins are expensive at scale.

### ODBC/JDBC Connections

Data pipelines connect to databases using:
- **JDBC**: Java Database Connectivity — standard for Java/Scala (Spark uses this)
- **ODBC**: Open Database Connectivity — cross-platform standard
- **SQLAlchemy**: Python's ORM + connection abstraction

```python
# SQLAlchemy connection string formats
postgresql = "postgresql+psycopg2://user:password@host:5432/dbname"
mysql = "mysql+pymysql://user:password@host:3306/dbname"
mssql = "mssql+pyodbc://user:password@host:1433/dbname?driver=ODBC+Driver+17+for+SQL+Server"
sqlite = "sqlite:///local.db"
```

---

## NoSQL Databases

NoSQL databases trade some SQL guarantees for scalability, flexibility, or specific access patterns.

### Key-Value Stores — HBase

HBase is a distributed key-value store built on top of HDFS. It scales to billions of rows.

**Architecture**:
- **Row key**: The primary access path — design carefully!
- **Column families**: Groups of columns stored together
- **Timestamp**: HBase keeps multiple versions of each cell

```
Row Key: "user_123|2024-01-15"
Column Family: "events"
  events:purchase = "49.99"
  events:login = "1"
  events:view = "15"
Column Family: "profile"
  profile:email = "user@example.com"
  profile:country = "US"
```

**Use HBase when**:
- Random read/write access to billions of rows
- Each row has millions of sparse columns
- Time-series data with key-based access
- Example: OpenTSDB (time series on HBase), user activity logs

**HBase vs Cassandra**:
- **HBase**: Strong consistency, HDFS-based, lower write throughput
- **Cassandra**: Eventual consistency, higher write throughput, more operational flexibility

### Document Stores — MongoDB

MongoDB stores data as JSON-like documents (BSON format).

```json
{
  "_id": "507f1f77bcf86cd799439011",
  "user_id": "user_123",
  "order": {
    "items": [
      {"product_id": "prod_001", "quantity": 2, "price": 19.99},
      {"product_id": "prod_002", "quantity": 1, "price": 49.99}
    ],
    "total": 89.97,
    "shipping_address": {
      "street": "123 Main St",
      "city": "Austin",
      "state": "TX"
    }
  },
  "created_at": "2024-01-15T10:30:00Z"
}
```

**Advantages**:
- Flexible schema — add fields without altering tables
- Nested documents avoid joins
- Natural fit for hierarchical data

**Disadvantages**:
- No JOINs (must denormalize or use `$lookup`)
- ACID only within single document (multi-document transactions added later)
- Not suited for complex analytical queries

### Elasticsearch — Search Engine + Document Store

Elasticsearch indexes JSON documents for full-text search and analytics.

**Architecture**:
- Distributed: data split across shards across cluster nodes
- Each shard is a Lucene index
- Replicas provide fault tolerance

**Use Elasticsearch for**:
- Full-text search (search a product catalog, document search)
- Log analytics (with Kibana — the "ELK Stack")
- Real-time analytics on event data
- Application performance monitoring (APM)

```python
from elasticsearch import Elasticsearch

es = Elasticsearch(['localhost:9200'])

# Index a document
es.index(index='products', body={
    'name': 'MacBook Pro 16"',
    'description': 'Apple laptop with M3 chip, excellent for data engineering',
    'price': 2499.99,
    'category': 'laptops'
})

# Full-text search
results = es.search(index='products', body={
    'query': {
        'multi_match': {
            'query': 'data engineering laptop',
            'fields': ['name', 'description']
        }
    }
})

# Aggregation
agg_result = es.search(index='products', body={
    'aggs': {
        'avg_price_by_category': {
            'terms': {'field': 'category.keyword'},
            'aggs': {'avg_price': {'avg': {'field': 'price'}}}
        }
    }
})
```

### Graph Databases — Neo4j

Graph databases store data as nodes and relationships. The relationships are first-class citizens.

**When to use Graph DBs**:
- Social networks (friends-of-friends queries)
- Recommendation engines
- Fraud detection (detect fraud rings)
- Knowledge graphs
- Network and IT infrastructure

**Cypher Query Language** (Neo4j):
```cypher
-- Find friends of friends who like the same movies
MATCH (user:User {name: 'Alice'})-[:FRIENDS_WITH]->(friend)-[:LIKES]->(movie:Movie)
WHERE NOT (user)-[:LIKES]->(movie)
RETURN movie.title, COUNT(friend) AS mutual_friends
ORDER BY mutual_friends DESC
LIMIT 10;

-- Create a relationship
MATCH (a:User {id: '123'}), (b:User {id: '456'})
CREATE (a)-[:FOLLOWS {since: date()}]->(b)
```

**Advantages of Neo4j**:
- Schema-free
- Traversing relationships is O(1) per hop (vs expensive JOINs in SQL)
- Cypher is intuitive for graph patterns

**Disadvantages**:
- Not suited for aggregations/OLAP
- Sharding not supported natively

### Column Databases — Cassandra

Cassandra is a distributed wide-column store designed for high write throughput and linear scalability.

**Key design principle**: Design your schema around your queries, not your entities.

```sql
-- Cassandra CQL
CREATE KEYSPACE user_events
WITH replication = {'class': 'NetworkTopologyStrategy', 'us-east': 3};

-- Table designed for query: "get last 100 events for user X"
CREATE TABLE user_events.events_by_user (
    user_id UUID,
    event_time TIMESTAMP,
    event_type TEXT,
    metadata MAP<TEXT, TEXT>,
    PRIMARY KEY (user_id, event_time)  -- user_id is partition key
) WITH CLUSTERING ORDER BY (event_time DESC)
  AND default_time_to_live = 2592000;   -- 30 day TTL

-- Query
SELECT * FROM user_events.events_by_user
WHERE user_id = 'd290f1ee-6c54-4b01-90e6-d701748f0851'
LIMIT 100;
```

**Use Cassandra when**:
- Extremely high write throughput (millions of writes/second)
- Time-series data with predictable access patterns
- Geographic distribution (multi-datacenter replication)
- Examples: Netflix (user activity), Discord (messages), Apple (device metrics)

### InfluxDB — Time Series Database

Optimized specifically for time-series data (metrics, IoT, monitoring).

```python
from influxdb_client import InfluxDBClient, Point
from influxdb_client.client.write_api import SYNCHRONOUS

client = InfluxDBClient(url="http://localhost:8086", token="my-token", org="myorg")
write_api = client.write_api(write_options=SYNCHRONOUS)

# Write metric
point = Point("cpu_usage") \
    .tag("host", "server-01") \
    .tag("region", "us-east") \
    .field("value", 78.5) \
    .time(datetime.utcnow())

write_api.write(bucket="metrics", record=point)

# Query with Flux language
query = '''
from(bucket: "metrics")
  |> range(start: -1h)
  |> filter(fn: (r) => r._measurement == "cpu_usage")
  |> filter(fn: (r) => r.host == "server-01")
  |> mean()
'''
result = client.query_api().query(query=query)
```

### Apache Druid — Real-Time OLAP

Druid is a column-oriented distributed data store for real-time analytics on event data.

**Key characteristics**:
- Sub-second queries on billions of rows
- Ingests from Kafka in real-time
- Pre-aggregates data at ingestion time
- Used by: Airbnb, Netflix, Twitter, Lyft

**vs ClickHouse**:
- Druid: Real-time ingestion from Kafka, complex topology
- ClickHouse: Better for batch-loaded data, simpler, faster for wide queries

---

## Data Warehouses

A data warehouse is an analytical database optimized for business intelligence queries.

### Snowflake

Snowflake is the leading cloud-native data warehouse with a unique multi-cluster architecture.

**Snowflake Architecture**:
```
Storage Layer:     S3/GCS/Azure Blob (columnar Parquet files)
Compute Layer:     Virtual Warehouses (independent, auto-scaling)
Cloud Services:    Query optimization, security, metadata
```

**Key Snowflake features**:

```sql
-- Time Travel: Query data as it was 1 day ago
SELECT * FROM orders AT(OFFSET => -86400);  -- 86400 seconds = 24 hours
SELECT * FROM orders BEFORE(STATEMENT => '01a8be32-0000-0000-0000-000000000000');

-- Zero-copy cloning: Instant clone of TB tables
CREATE TABLE orders_backup CLONE orders;

-- Data sharing: Share live data with other Snowflake accounts
CREATE SHARE analytics_share;
GRANT USAGE ON DATABASE analytics TO SHARE analytics_share;
ALTER SHARE analytics_share ADD ACCOUNTS = partner_account_name;

-- Snowpipe: Auto-ingest from S3 when files land
CREATE PIPE my_pipe AUTO_INGEST = TRUE AS
COPY INTO raw.events
FROM @my_s3_stage
FILE_FORMAT = (TYPE = PARQUET);

-- Semi-structured data
SELECT
    event_data:user_id::VARCHAR AS user_id,
    event_data:properties:amount::FLOAT AS amount
FROM raw_events;  -- Native JSON querying
```

### Google BigQuery

BigQuery is a serverless, fully-managed data warehouse. No cluster to manage.

```sql
-- Partitioned tables (reduce scan cost)
CREATE TABLE `project.dataset.events`
PARTITION BY DATE(event_timestamp)
CLUSTER BY user_id, event_type
AS SELECT * FROM raw_events;

-- Cost control: estimate query cost before running
SELECT * FROM `project.dataset.events`
WHERE DATE(event_timestamp) = '2024-01-15'
-- Check "Estimated bytes to be processed" before running

-- BigQuery ML — train ML models in SQL!
CREATE OR REPLACE MODEL `project.dataset.churn_model`
OPTIONS(model_type='logistic_reg', input_label_cols=['churned']) AS
SELECT * FROM training_data;

-- Scheduled queries
-- Via BigQuery UI or command line scheduler

-- BQML prediction
SELECT * FROM ML.PREDICT(MODEL `project.dataset.churn_model`,
    (SELECT * FROM user_features WHERE prediction_date = CURRENT_DATE))
```

### Amazon Redshift

Amazon Redshift is AWS's massively parallel processing (MPP) columnar data warehouse.

```sql
-- Distribution keys: control how data is distributed across nodes
CREATE TABLE orders (
    order_id BIGINT NOT NULL,
    user_id BIGINT NOT NULL DISTKEY,  -- Distribute by user_id (often used in joins)
    amount DECIMAL(10,2),
    created_at TIMESTAMP
) SORTKEY(created_at);                -- Sort key for range queries

-- Analyze query performance
EXPLAIN SELECT * FROM orders WHERE user_id = 12345;

-- Vacuum (reclaim space after deletes/updates)
VACUUM orders;
ANALYZE orders;

-- COPY from S3 (fastest way to load data)
COPY orders FROM 's3://bucket/orders/'
IAM_ROLE 'arn:aws:iam::ACCOUNT:role/RedshiftRole'
FORMAT AS PARQUET;
```

### Data Warehouse Best Practices

1. **Partition by date**: Every large fact table should be partitioned by a date column
2. **Cluster/sort keys**: Match your most common query predicates
3. **Data types matter**: Use correct types (TIMESTAMP not VARCHAR for dates)
4. **Avoid SELECT ***: Columnar DBs only read columns you select — be specific
5. **Use CTAS for bulk loads**: `CREATE TABLE new AS SELECT...` is faster than INSERT
6. **Statistics**: Keep stats up to date for query optimizer
7. **Views for business logic**: Abstract complex SQL behind views for consumers

---

## Data Lakes

A data lake is cheap, flexible raw data storage — the foundation of the modern data platform.

### AWS S3 as a Data Lake

```
s3://my-company-datalake/
├── raw/                           # Bronze layer: exact copies from sources
│   ├── crm/salesforce/
│   │   └── 2024/01/15/
│   │       └── contacts.json
│   ├── web/clickstream/
│   │   └── 2024/01/15/
│   │       └── events.json.gz
│   └── erp/orders/
│       └── 2024/01/15/
│           └── orders.parquet
├── processed/                     # Silver layer: cleaned, joined
│   ├── users/
│   └── orders/
└── curated/                       # Gold layer: business aggregations
    ├── daily_revenue/
    └── user_segments/
```

**S3 Best Practices**:
```python
import boto3

s3 = boto3.client('s3')

# Write with partitioning
key = f"raw/orders/year=2024/month=01/day=15/orders.parquet"
s3.upload_file('local_file.parquet', 'my-bucket', key)

# List objects with prefix
paginator = s3.get_paginator('list_objects_v2')
for page in paginator.paginate(Bucket='my-bucket', Prefix='raw/orders/year=2024/'):
    for obj in page.get('Contents', []):
        print(obj['Key'])
```

---

## Data Lakehouses — Open Table Formats

Open table formats bring ACID transactions, time travel, and schema enforcement to data lakes.

### Delta Lake

Created by Databricks, now open-sourced. The most popular open table format.

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .config("spark.jars.packages", "io.delta:delta-core_2.12:2.4.0") \
    .config("spark.sql.extensions", "io.delta.sql.DeltaSparkSessionExtension") \
    .getOrCreate()

# Write Delta table
df.write.format("delta").mode("overwrite").save("s3://bucket/delta/orders/")

# ACID merge (UPSERT)
from delta.tables import DeltaTable

delta_table = DeltaTable.forPath(spark, "s3://bucket/delta/orders/")
delta_table.alias("target").merge(
    source=new_df.alias("source"),
    condition="target.order_id = source.order_id"
).whenMatchedUpdate(set={
    "status": "source.status",
    "updated_at": "source.updated_at"
}).whenNotMatchedInsertAll().execute()

# Time travel
spark.read.format("delta") \
    .option("timestampAsOf", "2024-01-01") \
    .load("s3://bucket/delta/orders/")

# Version history
spark.sql("DESCRIBE HISTORY delta.`s3://bucket/delta/orders/`").show()

# Z-ordering (data skipping optimization)
spark.sql("""
    OPTIMIZE delta.`s3://bucket/delta/orders/`
    ZORDER BY (user_id, created_at)
""")
```

### Apache Iceberg

Apache Iceberg is the open standard supported by AWS Athena, Snowflake, Spark, Trino, and more.

```sql
-- Create Iceberg table via Spark SQL
CREATE TABLE catalog.db.orders (
    order_id BIGINT,
    user_id BIGINT,
    amount DECIMAL(10,2),
    created_at TIMESTAMP
) USING iceberg
PARTITIONED BY (days(created_at))
TBLPROPERTIES ('write.format.default' = 'parquet');

-- Time travel
SELECT * FROM catalog.db.orders FOR SYSTEM_TIME AS OF TIMESTAMP '2024-01-01 00:00:00';
SELECT * FROM catalog.db.orders FOR VERSION AS OF 1234;

-- Snapshots
SELECT * FROM catalog.db.orders.snapshots;

-- MERGE (upsert)
MERGE INTO catalog.db.orders t
USING new_orders s ON t.order_id = s.order_id
WHEN MATCHED THEN UPDATE SET t.status = s.status
WHEN NOT MATCHED THEN INSERT *;
```

**Iceberg advantages over Delta Lake**:
- Open standard (no vendor lock-in)
- Works natively with Athena, Glue, Trino, Hive
- Hidden partitioning (partition by transformed columns)
- Better support for partition evolution

### Apache Hudi

Apache Hudi (Hadoop Upserts Deletes and Incrementals) focuses on streaming upserts.

**Hudi table types**:
- **Copy-on-Write (CoW)**: Rewrites entire file on update — good for reads
- **Merge-on-Read (MoR)**: Stores deltas separately, merges on read — good for writes

**Use case**: CDC (Change Data Capture) pipelines where source systems produce inserts, updates, and deletes.

---

## Serialization Formats

The format you store data in has massive impact on performance and cost.

### Apache Parquet (Recommended for OLAP)

Columnar format — stores each column's data together. Huge performance win for analytics.

```
Row-oriented (CSV/JSON):        Column-oriented (Parquet):
| id | name  | amount |         id column:     [1, 2, 3, ...]
|  1 | Alice | 100    |  →      name column:   [Alice, Bob, Carol, ...]
|  2 | Bob   | 200    |         amount column: [100, 200, 300, ...]
|  3 | Carol | 300    |
```

**Why Parquet is faster for analytics**:
- Query `SELECT SUM(amount)` only reads the `amount` column (others ignored)
- Predicate pushdown: skip entire row groups where min/max don't match your filter
- Compression: similar values in a column compress far better than row data

**Parquet + Snappy compression** is the standard for big data storage.

```python
# Write Parquet with pandas
df.to_parquet(
    'output.parquet',
    engine='pyarrow',
    compression='snappy',
    index=False
)

# Write partitioned Parquet
df.to_parquet(
    's3://bucket/events/',
    engine='pyarrow',
    partition_cols=['year', 'month', 'day'],
    compression='snappy'
)
```

### Apache Avro (Recommended for Streaming)

Row-based format with schema embedded in the file. Ideal for Kafka + schema evolution.

```json
{
  "type": "record",
  "name": "UserEvent",
  "fields": [
    {"name": "user_id", "type": "string"},
    {"name": "event_type", "type": "string"},
    {"name": "amount", "type": ["null", "double"], "default": null},
    {"name": "timestamp", "type": {"type": "long", "logicalType": "timestamp-millis"}}
  ]
}
```

**Schema evolution**: Avro allows adding fields with defaults, making backward/forward compatibility easy.

### Apache ORC

Similar to Parquet but designed for Hive. Used in older Hadoop ecosystems.

### Protocol Buffers (Protobuf)

Google's binary serialization format. Faster and smaller than JSON, widely used in microservices.

### Format Comparison

| Format | Type | Use Case | Compression | Schema |
|--------|------|---------|------------|--------|
| CSV | Row | Simple exchange | None/Gzip | No |
| JSON | Row | APIs, logging | None/Gzip | No |
| Avro | Row | Kafka, streaming | Snappy/Deflate | Yes (embedded) |
| Parquet | Column | Analytics, warehousing | Snappy/Zstd | Yes |
| ORC | Column | Hive, HDFS | Zlib/Snappy | Yes |
| Protobuf | Row | Microservices APIs | No | Yes (.proto file) |

---

## Choosing the Right Storage

### Decision Framework

**Step 1: What are the access patterns?**
- Random reads/writes by key → Key-value store (Redis, HBase)
- Full-table scans and aggregations → Column store (Snowflake, BigQuery, ClickHouse)
- Search on text fields → Elasticsearch
- Graph traversal → Neo4j
- Append-only time series → InfluxDB, TimescaleDB

**Step 2: What's the scale?**
- GBs → PostgreSQL is fine
- 100s of GBs → Managed data warehouse (BigQuery, Snowflake)
- TBs–PBs → Data lake + query engine (S3 + Iceberg + Trino)

**Step 3: What are the consistency requirements?**
- Financial transactions → PostgreSQL (ACID)
- User analytics → BigQuery/Snowflake (eventual consistency OK)
- Real-time dashboards → ClickHouse or Apache Druid

**Step 4: What's your budget?**
- Cheapest storage: S3/GCS (cents per GB)
- Moderate cost: PostgreSQL on RDS
- Higher cost: Snowflake, BigQuery (but managed, no ops)

### Quick Reference: Use-Case → Storage

| Use Case | Best Choice |
|---------|------------|
| Production app database | PostgreSQL |
| User session/cache | Redis |
| Full-text search | Elasticsearch |
| IoT metrics | InfluxDB or TimescaleDB |
| Real-time analytics | ClickHouse or Apache Druid |
| Data warehouse (small-medium) | Snowflake or BigQuery |
| Data lake (petabytes) | S3/GCS + Iceberg |
| Graph/recommendation | Neo4j |
| High-write time series | Cassandra |
| ML feature store | Feast (on top of Redis + warehouse) |

---

*← [04 - Advanced Skills](04-advanced-skills.md) | [06 - Orchestration](06-orchestration.md) →*
