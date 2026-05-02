# 20 — Real-Time Analytics Systems

> **Guide by [Mano Harsha Sappa](https://www.linkedin.com/in/manoharshasappa/)** | [Portfolio](https://manoharshasappa.github.io/portfolio_ManoHarshaSappa/) | [Back to Index](../README.md)

---

## Contents

- [Real-Time vs Near-Real-Time vs Batch](#real-time-vs-near-real-time-vs-batch)
- [OLAP Engines for Real-Time Analytics](#olap-engines-for-real-time-analytics)
- [Apache Druid](#apache-druid)
- [ClickHouse Deep Dive](#clickhouse-deep-dive)
- [Apache Pinot](#apache-pinot)
- [Streaming SQL with Apache Flink](#streaming-sql-with-apache-flink)
- [Lambda vs Kappa Architecture](#lambda-vs-kappa-architecture)
- [Change Data Capture (CDC)](#change-data-capture-cdc)
- [Real-Time Dashboards & APIs](#real-time-dashboards--apis)
- [Architecture Patterns](#architecture-patterns)

---

## Real-Time vs Near-Real-Time vs Batch

Not all "real-time" is the same. Understand the spectrum:

| Pattern | Latency | Complexity | Use Cases |
|---------|---------|-----------|-----------|
| **Batch** | Hours to days | Low | Historical reports, nightly ETL |
| **Micro-batch** | Seconds to minutes | Medium | Spark Structured Streaming |
| **Near real-time** | Seconds | Medium-High | Kafka → ClickHouse dashboards |
| **Real-time** | Milliseconds | High | Fraud detection, gaming leaderboards |
| **Complex Event** | Microseconds | Very High | HFT, network monitoring |

### When Do You Actually Need Real-Time?

Before building real-time pipelines (which are 5–10x harder to maintain than batch), ask:

```
Does the business decision change if data is 1 hour old? → Maybe batch is fine
Does the business decision change if data is 5 minutes old? → Near-real-time
Does someone/something need to react in seconds? → Real-time
Does someone/something need to react in milliseconds? → Low-latency real-time
```

Real-time analytics use cases that are worth the complexity:
- Fraud detection (act before the transaction completes)
- Live dashboards for operations teams (ridesharing, food delivery)
- Personalization (serve a recommendation before the user scrolls)
- Live gaming leaderboards
- Real-time bidding in ad tech
- IoT and sensor monitoring

---

## OLAP Engines for Real-Time Analytics

Traditional warehouses (Snowflake, BigQuery) are optimized for batch queries. For real-time analytics at low latency, use purpose-built OLAP engines:

### Comparison

| Engine | Latency | Best For | Scale |
|--------|---------|----------|-------|
| **ClickHouse** | <100ms | Time-series, event analytics | Billions of rows |
| **Apache Druid** | <1 second | Time-series, real-time ingestion | Trillions of rows |
| **Apache Pinot** | <10ms | User-facing analytics, low latency | Hundreds of billions |
| **StarRocks** | <100ms | All-purpose, fast joins | Very large scale |
| **DuckDB** | <1ms | Local analytics, embedded | Single machine |
| **Redpanda** | streaming | Event streaming (Kafka drop-in) | |

---

## Apache Druid

**Apache Druid** is a real-time analytics database designed for fast slice-and-dice queries over large time-series datasets.

### Architecture

```
Data Sources (Kafka, S3, files)
         ↓
[INGESTION] — Real-time (Kafka) or Batch (S3)
         ↓
[MIDDLE MANAGERS] — Coordinate ingestion tasks
         ↓
[HISTORICAL NODES] — Serve old, indexed segments
[REALTIME NODES]  — Serve freshly ingested data
         ↓
[BROKER] — Routes queries to right nodes
         ↓
[ROUTER] — Load balances incoming queries
         ↓
Client (Superset, Grafana, API)
```

### Key Concepts

- **Datasource**: Like a table, but time-partitioned into segments
- **Segment**: A time chunk of data, compressed and indexed
- **Rollup**: Pre-aggregate data at ingestion to reduce storage and speed up queries
- **Dimension**: String columns used for filtering/grouping
- **Metric**: Numeric columns that are aggregated

### Ingestion from Kafka

```json
{
  "type": "kafka",
  "dataSchema": {
    "dataSource": "page_views",
    "timestampSpec": {
      "column": "timestamp",
      "format": "iso"
    },
    "dimensionsSpec": {
      "dimensions": ["user_id", "page", "country", "device_type"]
    },
    "metricsSpec": [
      {"type": "count", "name": "count"},
      {"type": "longSum", "name": "session_duration", "fieldName": "session_duration"}
    ],
    "granularitySpec": {
      "type": "uniform",
      "segmentGranularity": "HOUR",
      "queryGranularity": "MINUTE",
      "rollup": true
    }
  },
  "ioConfig": {
    "topic": "page_view_events",
    "consumerProperties": {
      "bootstrap.servers": "kafka:9092"
    }
  }
}
```

### Querying Druid

```sql
-- Real-time query: page views by country in last 5 minutes
SELECT
    country,
    SUM("count") AS page_views,
    SUM(session_duration) / SUM("count") AS avg_session_seconds
FROM page_views
WHERE __time >= CURRENT_TIMESTAMP - INTERVAL '5' MINUTE
GROUP BY country
ORDER BY page_views DESC
LIMIT 20
```

---

## ClickHouse Deep Dive

**ClickHouse** is a column-oriented OLAP database built for extremely fast analytical queries. Originally developed at Yandex, now widely used at Cloudflare, Uber, eBay, and many others.

### Why ClickHouse is Fast

1. **Column storage**: Only reads columns used in the query (not full rows)
2. **Vectorized execution**: Processes data in CPU-cache-friendly batches
3. **Data compression**: Up to 10x compression vs row stores
4. **Sparse indexes**: Skips irrelevant data blocks without full scan
5. **Async inserts**: Batches small writes for efficiency

### Installation

```bash
# Docker
docker run -d --name clickhouse-server \
  -p 8123:8123 -p 9000:9000 \
  --ulimit nofile=262144:262144 \
  clickhouse/clickhouse-server

# Connect
docker exec -it clickhouse-server clickhouse-client
```

### Creating Tables

```sql
-- ReplacingMergeTree: deduplicates rows with same primary key
CREATE TABLE events
(
    event_id     UUID,
    user_id      UInt64,
    event_type   LowCardinality(String),
    page         String,
    country      LowCardinality(String),
    device_type  LowCardinality(String),
    session_id   UUID,
    revenue      Decimal(18, 2),
    event_time   DateTime,
    date         Date MATERIALIZED toDate(event_time)
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_time)
ORDER BY (event_type, user_id, event_time)
TTL event_time + INTERVAL 2 YEAR DELETE;

-- AggregatingMergeTree: store pre-aggregated data
CREATE TABLE events_hourly_agg
(
    event_hour   DateTime,
    event_type   LowCardinality(String),
    country      LowCardinality(String),
    event_count  AggregateFunction(count, UInt64),
    revenue_sum  AggregateFunction(sum, Decimal(18, 2))
)
ENGINE = AggregatingMergeTree()
ORDER BY (event_hour, event_type, country);
```

### Materialized Views (Real-Time Aggregation)

```sql
-- Automatically aggregate on insert — zero-latency pre-aggregation
CREATE MATERIALIZED VIEW events_hourly_mv
TO events_hourly_agg
AS SELECT
    toStartOfHour(event_time) AS event_hour,
    event_type,
    country,
    countState() AS event_count,
    sumState(revenue) AS revenue_sum
FROM events
GROUP BY event_hour, event_type, country;
```

### Real-Time Kafka Ingestion

```sql
-- ClickHouse reads directly from Kafka
CREATE TABLE events_kafka_queue
(
    event_id   String,
    user_id    UInt64,
    event_type String,
    event_time DateTime,
    revenue    Decimal(18, 2)
)
ENGINE = Kafka
SETTINGS
    kafka_broker_list = 'kafka:9092',
    kafka_topic_list = 'events',
    kafka_group_name = 'clickhouse-consumer',
    kafka_format = 'JSONEachRow';

-- Materialized view moves data from queue to storage table
CREATE MATERIALIZED VIEW events_kafka_mv TO events AS
SELECT * FROM events_kafka_queue;
```

### Query Performance Tips

```sql
-- Use PREWHERE for coarse filtering before WHERE (much faster)
SELECT count()
FROM events
PREWHERE date >= today() - 7  -- fast: uses sparse index
WHERE event_type = 'purchase'  -- applied after PREWHERE

-- Use sampling for approximate results on massive datasets
SELECT
    country,
    countIf(event_type = 'purchase') / count() AS purchase_rate
FROM events SAMPLE 0.1  -- query 10% of data, 10x faster
WHERE date >= today() - 30

-- Avoid SELECT * — always specify columns
-- Bad:  SELECT * FROM events WHERE date = today()
-- Good: SELECT user_id, event_type, event_time FROM events WHERE date = today()
```

---

## Apache Pinot

**Apache Pinot** is a real-time OLAP store purpose-built for **user-facing analytics** at very low latency (< 10ms). LinkedIn and Uber built it for serving analytics directly to end users.

### Key Differentiators from ClickHouse/Druid

- Designed for **sub-10ms** P99 latency
- Supports **upserts** (real-time deduplication)
- **Star-tree index** for multi-dimensional fast aggregations
- Ideal for: product analytics dashboards shown to millions of users

### Use Cases

- LinkedIn's "Who viewed your profile" analytics
- Uber Eats restaurant analytics dashboard (served to restaurants in real-time)
- Twitter's live engagement metrics

### Schema Definition

```json
{
  "schemaName": "orders",
  "dimensionFieldSpecs": [
    {"name": "orderId", "dataType": "LONG"},
    {"name": "userId", "dataType": "LONG"},
    {"name": "restaurantId", "dataType": "LONG"},
    {"name": "status", "dataType": "STRING"},
    {"name": "country", "dataType": "STRING"}
  ],
  "metricFieldSpecs": [
    {"name": "orderAmount", "dataType": "DOUBLE"},
    {"name": "deliveryTime", "dataType": "DOUBLE"}
  ],
  "dateTimeFieldSpecs": [
    {
      "name": "orderTime",
      "dataType": "TIMESTAMP",
      "format": "1:MILLISECONDS:EPOCH",
      "granularity": "1:MINUTES"
    }
  ]
}
```

---

## Streaming SQL with Apache Flink

**Apache Flink** is the leading framework for **stateful stream processing**. It processes events one at a time (or in micro-windows) with exactly-once guarantees.

### Flink vs Spark Streaming

| | Flink | Spark Streaming |
|--|-------|----------------|
| Processing model | True streaming | Micro-batch |
| Latency | Milliseconds | Seconds |
| State management | Built-in, scalable | Limited |
| Exactly-once | Yes | Yes (with checkpoints) |
| Maturity | Mature | Very mature |

### Flink SQL Example

```sql
-- Flink SQL: real-time revenue aggregation by minute
CREATE TABLE orders_kafka (
    order_id     BIGINT,
    user_id      BIGINT,
    order_amount DOUBLE,
    status       STRING,
    event_time   TIMESTAMP(3),
    WATERMARK FOR event_time AS event_time - INTERVAL '5' SECOND
) WITH (
    'connector' = 'kafka',
    'topic' = 'orders',
    'properties.bootstrap.servers' = 'kafka:9092',
    'format' = 'json'
);

CREATE TABLE revenue_by_minute (
    window_start  TIMESTAMP(3),
    window_end    TIMESTAMP(3),
    order_count   BIGINT,
    total_revenue DOUBLE
) WITH (
    'connector' = 'jdbc',
    'url' = 'jdbc:postgresql://...',
    'table-name' = 'revenue_by_minute'
);

-- Tumbling window aggregation
INSERT INTO revenue_by_minute
SELECT
    TUMBLE_START(event_time, INTERVAL '1' MINUTE) AS window_start,
    TUMBLE_END(event_time, INTERVAL '1' MINUTE)   AS window_end,
    COUNT(*) AS order_count,
    SUM(order_amount) AS total_revenue
FROM orders_kafka
WHERE status = 'completed'
GROUP BY TUMBLE(event_time, INTERVAL '1' MINUTE);
```

### Stateful Operations

```python
from pyflink.datastream import StreamExecutionEnvironment
from pyflink.datastream.functions import KeyedProcessFunction
from pyflink.common.state import ValueStateDescriptor

class SessionAggregator(KeyedProcessFunction):
    def open(self, runtime_context):
        # State persists across events for the same key
        self.session_start = runtime_context.get_state(
            ValueStateDescriptor("session_start", Types.LONG())
        )
        self.event_count = runtime_context.get_state(
            ValueStateDescriptor("event_count", Types.LONG())
        )

    def process_element(self, value, ctx, out):
        if self.session_start.value() is None:
            self.session_start.update(value.event_time)

        self.event_count.update((self.event_count.value() or 0) + 1)

        # Emit session after 30 minutes of inactivity
        ctx.timer_service().register_event_time_timer(
            value.event_time + 30 * 60 * 1000
        )

    def on_timer(self, timestamp, ctx, out):
        # Session ended — emit aggregated result
        out.collect({
            "user_id": ctx.get_current_key(),
            "session_start": self.session_start.value(),
            "session_end": timestamp,
            "event_count": self.event_count.value()
        })
        self.session_start.clear()
        self.event_count.clear()
```

---

## Lambda vs Kappa Architecture

### Lambda Architecture (Traditional)

```
Data Sources
    ↓         ↓
[BATCH]   [STREAM]
Hadoop    Kafka+Flink
Spark       ↓
    ↓    Speed Layer
Batch Layer (hot data)
(historical)
    ↓         ↓
    └────┬────┘
    Serving Layer
    (merges batch + stream)
```

**Problem**: Two codebases doing the same thing. Hard to maintain consistency.

### Kappa Architecture (Modern)

```
Data Sources
     ↓
[KAFKA] — Event log (source of truth, retained long-term)
     ↓           ↓
[FLINK]     [REPROCESS]
Real-time    (reprocess history from Kafka when logic changes)
     ↓
[CLICKHOUSE/DRUID]
Serving Layer
```

**Benefit**: One codebase. Reprocess from Kafka when the logic changes. Simpler to maintain.

### When to Use Which

| Scenario | Recommendation |
|----------|---------------|
| Simple batch + some real-time | Lambda with Spark Streaming |
| Pure streaming, complex state | Kappa with Flink |
| Small team, limited resources | Near-real-time batch (Spark 5min micro-batch) |
| Need sub-second latency | Kappa with Flink + Pinot/ClickHouse |

---

## Change Data Capture (CDC)

**CDC** tracks row-level changes (INSERT, UPDATE, DELETE) in a source database and streams them to downstream systems.

### Why CDC?

- Replicate production databases to analytics warehouse without full table scans
- Minimal load on source database
- Near-real-time replication (seconds, not hours)
- Capture deletes (which full dumps miss)

### CDC Tools

| Tool | Source | Best For |
|------|--------|----------|
| **Debezium** | MySQL, Postgres, MongoDB, etc. | Open-source, Kafka integration |
| **AWS DMS** | Most databases | AWS-native, managed |
| **Fivetran** | Many sources | No-code, managed |
| **Airbyte** | 300+ sources | Open-source, self-hosted |
| **Maxwell** | MySQL only | Simple MySQL CDC |

### Debezium + Kafka Setup

```json
// Debezium Postgres connector configuration
{
  "name": "postgres-orders-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "postgres",
    "database.port": "5432",
    "database.user": "debezium",
    "database.password": "secret",
    "database.dbname": "production",
    "database.server.name": "prod_db",
    "table.include.list": "public.orders,public.users",
    "plugin.name": "pgoutput",
    "publication.name": "debezium_pub",
    "slot.name": "debezium_slot",
    "transforms": "unwrap",
    "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState"
  }
}
```

CDC events look like:

```json
{
  "before": {"order_id": 123, "status": "pending", "amount": 99.99},
  "after":  {"order_id": 123, "status": "completed", "amount": 99.99},
  "op": "u",
  "ts_ms": 1700000000000,
  "source": {"table": "orders", "schema": "public"}
}
```

---

## Real-Time Dashboards & APIs

### Superset + ClickHouse Real-Time Dashboard

```
ClickHouse (stores events, has materialized views for pre-aggregation)
    ↓
Apache Superset (dashboards with auto-refresh every 30 seconds)
    ↓
Business users see live metrics
```

```python
# FastAPI endpoint serving real-time metrics from ClickHouse
from fastapi import FastAPI
from clickhouse_driver import Client

app = FastAPI()
ch = Client(host='clickhouse', database='analytics')

@app.get("/api/metrics/live")
def get_live_metrics():
    result = ch.execute("""
        SELECT
            event_type,
            count() AS event_count,
            sum(revenue) AS total_revenue
        FROM events
        WHERE event_time >= now() - INTERVAL 5 MINUTE
        GROUP BY event_type
        ORDER BY event_count DESC
    """)
    return [
        {"event_type": row[0], "count": row[1], "revenue": float(row[2])}
        for row in result
    ]

@app.get("/api/metrics/dashboard")
def get_dashboard_data():
    # Multiple parallel queries
    ...
```

---

## Architecture Patterns

### Pattern 1: Simple Near-Real-Time (Small Team)

```
PostgreSQL/MySQL
     ↓ (Airbyte every 5 min)
Snowflake/BigQuery
     ↓ (dbt every 15 min)
Analytics Tables
     ↓
Superset/Metabase Dashboard
```
**Good for**: Most analytics use cases, <15 minute acceptable latency

### Pattern 2: Kafka + ClickHouse (Medium Scale)

```
Application → Kafka → ClickHouse (Materialized Views)
                                   ↓
                              Superset / API
```
**Good for**: Event analytics, product dashboards, operations

### Pattern 3: Full Streaming (Large Scale)

```
Applications
    ↓
Kafka (Event Streaming)
    ↓            ↓
Flink       Raw events → S3 (data lake)
(Processing)
    ↓
ClickHouse / Pinot / Druid
    ↓
User-facing dashboards / APIs
```
**Good for**: Real-time user-facing analytics at scale

---

*Previous: [19 — Data Governance](19-data-governance.md) | Next: [21 — DataOps & CI/CD](21-dataops-cicd.md) →*
