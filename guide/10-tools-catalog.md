# Tools Catalog: 200+ Data Engineering Tools

**Author:** Mano Harsha Sappa | [LinkedIn](https://www.linkedin.com/in/manoharshasappa/) | [Portfolio](https://manoharshasappa.github.io/portfolio_ManoHarshaSappa/) | [Email](mailto:sappamanoharsha@gmail.com)

---

## Contents

1. [Databases — Relational](#1-databases--relational)
2. [Databases — Key-Value](#2-databases--key-value)
3. [Databases — Column Store](#3-databases--column-store)
4. [Databases — Document](#4-databases--document)
5. [Databases — Graph](#5-databases--graph)
6. [Databases — Time Series](#6-databases--time-series)
7. [Databases — Distributed & OLAP](#7-databases--distributed--olap)
8. [Data Ingestion & Integration](#8-data-ingestion--integration)
9. [Change Data Capture (CDC)](#9-change-data-capture-cdc)
10. [File Systems & Object Storage](#10-file-systems--object-storage)
11. [Serialization Formats](#11-serialization-formats)
12. [Stream Processing](#12-stream-processing)
13. [Batch Processing](#13-batch-processing)
14. [Data Lake Management (Lakehouse)](#14-data-lake-management-lakehouse)
15. [Orchestration & Workflow](#15-orchestration--workflow)
16. [Data Transformation (dbt & SQL)](#16-data-transformation-dbt--sql)
17. [Data Quality & Testing](#17-data-quality--testing)
18. [Data Governance & Catalog](#18-data-governance--catalog)
19. [Business Intelligence & Visualization](#19-business-intelligence--visualization)
20. [Messaging Infrastructure](#20-messaging-infrastructure)
21. [Monitoring & Observability](#21-monitoring--observability)
22. [Data Comparison & Profiling](#22-data-comparison--profiling)
23. [Infrastructure & Containerization](#23-infrastructure--containerization)
24. [Public Datasets](#24-public-datasets)
25. [Company Tool Ecosystem Map](#25-company-tool-ecosystem-map)

---

## 1. Databases — Relational

The backbone of structured data. These support ACID transactions and SQL.

| Tool | Description | Best For |
|------|-------------|----------|
| **PostgreSQL** | World's most advanced open-source RDBMS with rich extensions (pgvector, PostGIS) | General purpose, analytics, ML workloads |
| **MySQL** | World's most popular open-source RDBMS | Web apps, transactional OLTP |
| **MariaDB** | Drop-in MySQL replacement with extra features | MySQL migration, community-first |
| **TiDB** | Distributed NewSQL compatible with MySQL protocol | Horizontal scaling + MySQL compatibility |
| **Amazon RDS** | Managed relational DB on AWS (MySQL, PostgreSQL, SQL Server, Oracle) | Cloud-native OLTP |
| **Crate.IO** | Scalable SQL with NoSQL goodies (JSON, geospatial) | Log analytics, IoT data |
| **RQLite** | Replicated SQLite using Raft consensus | Lightweight distributed storage |
| **Percona XtraBackup** | Online hot backup for MySQL/MariaDB | Zero-downtime DB backups |

**When to use PostgreSQL:** Always start here. 80% of DE use cases fit PostgreSQL for OLTP/staging.

---

## 2. Databases — Key-Value

Ultra-fast lookups with O(1) complexity. Trade structure for speed.

| Tool | Description | Best For |
|------|-------------|----------|
| **Redis** | BSD-licensed in-memory KV store with Pub/Sub, streams, and data structures | Caching, session store, rate limiting, leaderboards |
| **AWS DynamoDB** | Managed NoSQL with single-digit millisecond latency at any scale | Serverless apps, IoT event data |
| **Riak** | Distributed database with high availability across multiple servers | Fault-tolerant distributed KV |
| **SSDB** | High-performance alternative to Redis with more data structures | Redis replacement with disk persistence |
| **IonDB** | KV store for microcontroller and IoT applications | Edge computing, embedded devices |

---

## 3. Databases — Column Store

Optimized for analytical queries that read few columns across many rows.

| Tool | Description | Best For |
|------|-------------|----------|
| **Apache Cassandra** | Highly scalable column-family store with no single point of failure | Write-heavy workloads, time-series at scale |
| **ScyllaDB** | Cassandra-compatible built with C++/Seastar, 10x faster | High-throughput Cassandra replacement |
| **HBase** | Hadoop's distributed wide-column store (BigTable clone) | Petabyte-scale sparse data on HDFS |
| **AWS Redshift** | Petabyte-scale managed columnar data warehouse | Enterprise analytical workloads on AWS |
| **ClickHouse** | Fastest open-source OLAP columnar database | Real-time analytics, log analysis |
| **Vertica** | Distributed MPP columnar DB with extensive analytics SQL | Enterprise-grade analytics |

**Cassandra key decision:** Choose it when you need linear write scalability with tunable consistency and can model data around your queries (denormalization).

```sql
-- Cassandra partition strategy example
CREATE TABLE user_events (
    user_id UUID,
    event_date DATE,
    event_time TIMESTAMP,
    event_type TEXT,
    payload TEXT,
    PRIMARY KEY ((user_id, event_date), event_time)
) WITH CLUSTERING ORDER BY (event_time DESC)
  AND default_time_to_live = 7776000; -- 90 days TTL
```

---

## 4. Databases — Document

Schema-flexible JSON documents. Great for heterogeneous data.

| Tool | Description | Best For |
|------|-------------|----------|
| **MongoDB** | Most popular document database with rich query language | Content management, catalogs, user profiles |
| **Elasticsearch** | Distributed search and analytics engine (inverted index) | Full-text search, log analytics, APM |
| **Couchbase** | Highest-performance NoSQL distributed document store | Mobile sync, gaming, financial services |
| **RethinkDB** | Real-time web with push notifications on query changes | Real-time collaborative apps |
| **RavenDB** | Fully transactional NoSQL document database | .NET-heavy shops wanting ACID docs |

---

## 5. Databases — Graph

Model relationships as first-class citizens. O(1) traversal vs O(n) joins.

| Tool | Description | Best For |
|------|-------------|----------|
| **Neo4j** | World's leading graph database with Cypher query language | Fraud detection, recommendation engines, knowledge graphs |
| **ArangoDB** | Multi-model: documents + graphs + key-value in one DB | Mixed data models without multiple DBs |
| **OrientDB** | 2nd gen distributed graph with document flexibility | Graph + relational combined queries |
| **ArcadeDB** | Multi-model with native graph, document, key-value, vector | Modern multi-paradigm workloads |
| **Titan** | Scalable graph optimized for hundreds of billions of edges | Massive graph analytics |

```cypher
-- Neo4j: find friends-of-friends who share interests
MATCH (user:User {id: 'U123'})-[:FRIEND]->(friend)-[:FRIEND]->(fof)
WHERE NOT (user)-[:FRIEND]->(fof)
AND fof <> user
WITH fof, COUNT(friend) AS mutual_friends
MATCH (fof)-[:LIKES]->(interest)<-[:LIKES]-(user)
RETURN fof.name, mutual_friends, COLLECT(interest.name) AS shared_interests
ORDER BY mutual_friends DESC
LIMIT 10
```

---

## 6. Databases — Time Series

Optimized for time-stamped metrics, events, and sensor data.

| Tool | Description | Best For |
|------|-------------|----------|
| **InfluxDB** | Purpose-built time series with Flux query language | DevOps metrics, IoT sensor data |
| **TimescaleDB** | PostgreSQL extension for time-series (best of both worlds) | Time-series + relational joins in one DB |
| **QuestDB** | Relational column-oriented DB for real-time time series analytics | High-frequency trading, financial data |
| **Apache Druid** | Column-oriented OLAP ideal for interactive analytics | Sub-second queries on event streams |
| **OpenTSDB** | Scalable distributed time series on HBase | Monitoring at Hadoop scale |
| **kairosdb** | Fast scalable time series on Cassandra | Alternative to OpenTSDB |
| **Heroic** | Spotify's time series DB on Cassandra + Elasticsearch | Music streaming metrics at Spotify scale |

**TimescaleDB example:**
```sql
-- Create a hypertable (auto-partitioned by time)
SELECT create_hypertable('sensor_readings', 'time');

-- Continuous aggregate for hourly rollups
CREATE MATERIALIZED VIEW sensor_hourly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', time) AS bucket,
    sensor_id,
    AVG(temperature) AS avg_temp,
    MAX(temperature) AS max_temp,
    MIN(temperature) AS min_temp
FROM sensor_readings
GROUP BY bucket, sensor_id;
```

---

## 7. Databases — Distributed & OLAP

Modern analytical databases built for scale.

| Tool | Description | Best For |
|------|-------------|----------|
| **DuckDB** | In-process OLAP — zero dependencies, runs in Python/R/JS | Local analytics, data science notebooks, "SQLite for analytics" |
| **Apache Pinot** | Realtime distributed OLAP datastore built by LinkedIn | User-facing analytics at millisecond latency |
| **StarRocks** | MPP analytical DB — fast querying of both streaming and batch | Real-time + historical analytics |
| **Apache Kylin** | Distributed OLAP cube on Hadoop/Spark | Pre-aggregated cube analytics |
| **GreenPlum** | Advanced open source MPP data warehouse on PostgreSQL | Enterprise data warehouse |
| **Dremio** | Data lake engine with Apache Arrow query acceleration | Self-service lakehouse querying |

```python
# DuckDB: analyze Parquet files without loading into memory
import duckdb

conn = duckdb.connect()
result = conn.execute("""
    SELECT
        date_trunc('month', order_date) AS month,
        SUM(revenue) AS total_revenue,
        COUNT(DISTINCT customer_id) AS unique_customers
    FROM read_parquet('s3://my-bucket/orders/*.parquet')
    WHERE order_date >= '2024-01-01'
    GROUP BY 1
    ORDER BY 1
""").df()
```

---

## 8. Data Ingestion & Integration

Moving data from sources to destinations reliably.

| Tool | Description | Type | Best For |
|------|-------------|------|----------|
| **Apache Kafka** | Distributed commit log / event streaming platform | Message broker | Real-time event pipelines, CDC |
| **Airbyte** | 300+ source connectors, open-source EL(T) | ELT | Replacing Fivetran for custom connectors |
| **Fivetran** | Managed ELT with zero-maintenance connectors | Managed ELT | Enterprise plug-and-play ingestion |
| **dlt** | Python library for building data pipelines in notebooks/Airflow | Library | Lightweight Python-first pipelines |
| **Meltano** | CLI + code-first ELT using Singer spec | CLI ELT | Singer tap/target ecosystem |
| **Apache Sqoop** | Bulk data transfer between Hadoop and RDBMS | Batch ingestion | HDFS ↔ relational DB bulk loads |
| **Apache Gobblin** | Universal ingestion framework from LinkedIn | Framework | Multi-source Hadoop ingestion |
| **Apache NiFi** | Visual dataflow automation with 300+ processors | Visual ETL | Complex routing, IoT data flows |
| **Fluentd** | Unified logging layer — collect, transform, route logs | Log collector | Centralized logging |
| **Embulk** | Bulk data loader across databases and cloud services | Batch loader | Database migrations |
| **Apache Pulsar** | Distributed pub-sub messaging with native multi-tenancy | Message broker | Kafka alternative with better geo-replication |
| **RabbitMQ** | Classic AMQP message broker | Message broker | Task queues, microservice comms |
| **AWS Kinesis** | Managed real-time data streaming on AWS | Managed streaming | AWS-native event streaming |
| **Sling** | CLI tool for moving data between databases and storage | CLI | Fast DB-to-DB or DB-to-cloud copy |
| **Estuary Flow** | No/low-code batch + real-time ingestion | Managed | Both batch and streaming in one platform |
| **Nakadi** | RESTful API on top of Kafka-like queues (Zalando) | REST Kafka proxy | HTTP-first event publishing |
| **ingestr** | Single-command CLI to copy data between 50+ sources | CLI | Quick one-off data copies |
| **Artie** | Real-time ingestion via CDC | Managed CDC | Sub-minute latency from DB to warehouse |

**Choosing an ingestion tool:**
```
Need real-time?  → Kafka (self-managed) or Kinesis (AWS managed)
Need 300+ sources with no code? → Fivetran (paid) or Airbyte (open-source)
Building in Python? → dlt
Need CDC from Postgres/MySQL? → Debezium
Need batch DB transfers? → Sqoop (Hadoop) or Sling (cloud)
```

---

## 9. Change Data Capture (CDC)

Capture every INSERT, UPDATE, DELETE from your operational database in real time.

| Tool | Description | Sources |
|------|-------------|---------|
| **Debezium** | Industry-standard CDC using database transaction logs | MySQL, PostgreSQL, MongoDB, SQL Server, Oracle |
| **Maxwell's Daemon** | MySQL-to-JSON Kafka producer — simple and battle-tested | MySQL |
| **AWS DMS** | Managed database migration + continuous replication | 20+ sources to 20+ targets |
| **Striim** | Enterprise real-time CDC and streaming integration | Oracle, SQL Server, MySQL, mainframe |

```python
# Debezium connector config (Kafka Connect)
debezium_config = {
    "name": "postgres-cdc-connector",
    "config": {
        "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
        "database.hostname": "postgres",
        "database.port": "5432",
        "database.user": "debezium",
        "database.password": "secret",
        "database.dbname": "ecommerce",
        "database.server.name": "prod",
        "table.include.list": "public.orders,public.users",
        "plugin.name": "pgoutput",
        "slot.name": "debezium_slot",
        "publication.name": "dbz_publication",
        "topic.prefix": "prod"
    }
}
# Produces events to topics: prod.public.orders, prod.public.users
```

---

## 10. File Systems & Object Storage

Where your data physically lives at scale.

| Tool | Description | Best For |
|------|-------------|----------|
| **AWS S3** | Industry-standard object storage — 99.999999999% durability | Cloud data lake foundation |
| **HDFS** | Hadoop Distributed File System — 3x replication on commodity hardware | On-premise big data |
| **MinIO** | S3-compatible object storage you can self-host | Private cloud / on-prem S3 |
| **Alluxio** | Memory-centric distributed storage tier — caches HDFS/S3 in memory | Accelerate Spark/Presto over slow storage |
| **CEPH** | Unified distributed storage (object + block + file) | On-prem multi-protocol storage |
| **JuiceFS** | Cloud-native file system over object storage | POSIX-compatible access to S3 |
| **SeaweedFS** | Simple, massively scalable distributed file system | Billions of small files |
| **GlusterFS** | Scale-out network-attached storage | NAS replacement |
| **LizardFS** | Software-defined distributed file system | Geo-redundant file storage |

**S3 lifecycle policy example:**
```json
{
  "Rules": [{
    "ID": "archive-old-data",
    "Status": "Enabled",
    "Transitions": [
      {"Days": 30, "StorageClass": "STANDARD_IA"},
      {"Days": 90, "StorageClass": "GLACIER"},
      {"Days": 365, "StorageClass": "DEEP_ARCHIVE"}
    ],
    "NoncurrentVersionExpiration": {"NoncurrentDays": 90}
  }]
}
```

---

## 11. Serialization Formats

How data is encoded for storage and transmission. This choice dramatically impacts performance.

| Format | Type | Schema | Compression | Read Speed | Write Speed | Best For |
|--------|------|--------|-------------|------------|-------------|----------|
| **Apache Parquet** | Columnar | Optional | Excellent (Snappy/Zstd) | Very Fast (column skip) | Moderate | Analytics, data lakes, Spark |
| **Apache ORC** | Columnar | Embedded | Excellent | Very Fast | Fast | Hive workloads |
| **Apache Avro** | Row | Required (schema evolution) | Good | Moderate | Very Fast | Kafka messages, CDC events |
| **Protocol Buffers** | Row | Required | Good | Fast | Fast | gRPC APIs, microservices |
| **Apache Arrow** | In-memory columnar | Optional | N/A (in-memory) | Blazing | Blazing | In-process analytics, DuckDB, Pandas |
| **JSON** | Row | None | Poor | Slow | Slow | APIs, debugging, small data |
| **Apache Thrift** | Row | Required | Good | Fast | Fast | Cross-language RPC |
| **MessagePack** | Row | None | Good | Fast | Fast | JSON replacement in binary |
| **FlatBuffers** | Row | Required | None (zero-copy) | Instant | Moderate | Games, mobile, zero-copy access |
| **Cap'n Proto** | Row | Required | None (zero-copy) | Instant | Instant | High-performance IPC |

**Format decision guide:**
```
Store analytics data on disk → Parquet (default choice)
Send events through Kafka → Avro (with Schema Registry)
Build a gRPC microservice → Protocol Buffers
Do in-memory analytics → Apache Arrow
Expose a REST API → JSON
Need Hive compatibility → ORC
```

**Parquet vs Avro deep dive:**
- **Parquet**: Column skip reads 1 column from 1000-column table in one I/O. Great for `SELECT revenue, date FROM orders WHERE date > '2024-01-01'`
- **Avro**: Schema evolution allows adding/removing fields without breaking consumers. Great for event streams where schema changes over time

---

## 12. Stream Processing

Processing data as it arrives, with low latency.

| Tool | Description | Latency | Exactly Once | Best For |
|------|-------------|---------|--------------|----------|
| **Apache Flink** | True stream processing with event time, watermarks, stateful ops | Milliseconds | Yes | Complex streaming ETL, CEP |
| **Apache Kafka Streams** | Client library — stream processing inside Kafka | Milliseconds | Yes | Kafka-native stateful processing |
| **Apache Spark Structured Streaming** | Micro-batch streaming on Spark engine | Seconds | Yes | Teams already on Spark |
| **Apache Beam** | Unified batch + streaming model (runs on Flink, Spark, Dataflow) | Configurable | Yes | Portable pipelines |
| **Apache Storm** | Original low-latency distributed streaming | Milliseconds | At-least-once | Legacy real-time systems |
| **Apache Samza** | Stateful stream processing backed by Kafka (LinkedIn) | Milliseconds | At-least-once | Kafka-coupled workloads |
| **Faust** | Python stream processing library (asyncio + Kafka) | Milliseconds | At-least-once | Python-first Kafka consumers |
| **Pathway** | Python ETL with Rust runtime, 300+ sources | Milliseconds | Yes | AI pipeline data freshness |
| **RisingWave** | PostgreSQL-compatible streaming database | Milliseconds | Yes | SQL-first streaming analytics |
| **SwimOS** | Framework for real-time streaming apps | Milliseconds | Yes | IoT and agent-based streaming |

**Flink stateful processing example:**
```python
from pyflink.datastream import StreamExecutionEnvironment
from pyflink.datastream.window import TumblingEventTimeWindows
from pyflink.common.time import Time

env = StreamExecutionEnvironment.get_execution_environment()

stream = (
    env.from_source(kafka_source, watermark_strategy, "Kafka Source")
    .key_by(lambda x: x["user_id"])
    .window(TumblingEventTimeWindows.of(Time.minutes(5)))
    .aggregate(RevenueAggregator())
)

stream.sink_to(kafka_sink)
env.execute("User Revenue 5-Minute Windows")
```

---

## 13. Batch Processing

Processing large datasets in scheduled jobs.

| Tool | Description | Language | Best For |
|------|-------------|----------|----------|
| **Apache Spark** | Unified analytics engine — batch, streaming, ML, graph | Python/Scala/Java/R | Everything at scale |
| **Hadoop MapReduce** | Original batch processing on HDFS | Java | Legacy Hadoop ecosystem |
| **Apache Tez** | DAG-based execution framework above YARN | Java | Replacing MapReduce for Hive |
| **AWS EMR** | Managed Hadoop/Spark on EC2 spot instances | Any | Elastic batch on AWS |
| **Apache Hive** | SQL-on-Hadoop using MapReduce/Tez | SQL | SQL users on HDFS data |
| **Presto / Trino** | Distributed SQL query engine across heterogeneous sources | SQL | Ad-hoc cross-source queries |
| **Apache Drill** | Schema-free SQL on Hadoop, NoSQL, cloud storage | SQL | Self-describing data exploration |
| **H2O.ai** | Fast scalable ML API (AutoML) | Python/R | AutoML on large datasets |
| **Apache Mahout** | Scalable ML algorithms on Spark | Scala | Distributed ML at Hadoop scale |
| **Spark MLlib** | Spark's built-in ML library | Python/Scala | ML within existing Spark pipelines |

---

## 14. Data Lake Management (Lakehouse)

Open table formats that bring database features (ACID, schema evolution, time travel) to your data lake.

| Tool | Description | Key Features | Created By |
|------|-------------|--------------|-----------|
| **Delta Lake** | ACID transactions on object storage | Time travel, MERGE, Z-ORDER clustering, Change Data Feed | Databricks |
| **Apache Iceberg** | High-performance table format for huge analytic tables | Hidden partitioning, partition evolution, snapshot isolation | Netflix |
| **Apache Hudi** | Transactional data lake with upsert support | Record-level indexing, incremental queries, MOR/COW tables | Uber |
| **lakeFS** | Git-like versioning for your data lake | Branch, commit, merge, rollback your data | Treeverse |
| **Project Nessie** | Transactional catalog with Git semantics for Iceberg | Multi-table transactions across Iceberg tables | Dremio |
| **Apache Gravitino** | Unified metadata management for lakes, warehouses, catalogs | Cross-platform metadata federation | Apache |

**Lakehouse comparison:**
```
Delta Lake:  Best integration with Databricks and Spark ecosystem
Iceberg:     Most engine-agnostic (Spark, Flink, Trino, Hive all support it)
Hudi:        Best for high-frequency upserts (streaming writes, CDC)

All three support:
✓ ACID transactions
✓ Schema evolution
✓ Time travel / snapshots
✓ Partition evolution
✓ Streaming + batch reads
```

---

## 15. Orchestration & Workflow

Scheduling, coordinating, and monitoring your data pipelines.

| Tool | Description | Paradigm | Best For |
|------|-------------|----------|----------|
| **Apache Airflow** | Dominant Python-based DAG scheduler | Code (Python DAGs) | Complex dependency graphs, mature ecosystem |
| **Prefect** | Modern workflow with dynamic task mapping, cloud observability | Code (Python flows) | Python-native teams wanting better DX than Airflow |
| **Dagster** | Asset-centric orchestrator with built-in lineage | Asset definitions | Data assets + software-defined assets |
| **Kestra** | Event-driven, language-agnostic declarative scheduler | YAML flows | Multi-language teams, no-code users |
| **Mage** | Interactive pipeline builder with Jupyter-like UI | GUI + Code | Data scientists building pipelines |
| **Luigi** | Spotify's Python pipeline framework (task dependencies) | Code (Python tasks) | Simple dependency resolution |
| **Azkaban** | LinkedIn's workflow scheduler for Hadoop jobs | Config files | Hadoop-heavy shops |
| **Hamilton** | Lightweight DAG library for Python functions | Decorator DAG | Modular Python transformation DAGs |
| **SQLMesh** | SQL transformation framework with CI/CD and environments | SQL + Python | dbt alternative with built-in state management |
| **Bruin** | End-to-end pipeline tool: ingest + transform + quality | YAML + SQL | All-in-one pipeline with VS Code extension |

**Orchestrator selection guide:**
```
Starting out / most popular:     → Apache Airflow
Want modern Python DX:           → Prefect
Need asset lineage built-in:     → Dagster
Multi-language / YAML first:     → Kestra
SQL + CI/CD environments:        → SQLMesh
All-in-one (ingest + transform): → Bruin
```

---

## 16. Data Transformation (dbt & SQL)

Transforming raw data into clean, documented, tested analytical models.

| Tool | Description | Best For |
|------|-------------|----------|
| **dbt Core** | Open-source SQL transformation framework with tests, docs, lineage | Analytics engineering standard |
| **dbt Cloud** | Managed dbt with scheduler, CI/CD, IDE | Enterprise dbt deployments |
| **SQLMesh** | dbt alternative with virtual environments and column-level lineage | Teams wanting state-aware CI |
| **Dataform** | Google's SQL transformation framework (now part of BigQuery) | BigQuery-native transformations |
| **Census** | Reverse ETL — sync warehouse data to SaaS apps via SQL | Operational analytics, CRM sync |
| **Multiwoven** | Open-source reverse ETL and data activation | Open-source Census alternative |

---

## 17. Data Quality & Testing

Ensuring your data is accurate, complete, fresh, and consistent.

| Tool | Description | Approach |
|------|-------------|----------|
| **Great Expectations** | Define "expectations" as rules; validate data at pipeline boundaries | Rule-based assertions |
| **Soda Core** | SQL-based data quality checks with YAML config | YAML quality contracts |
| **dbt Tests** | Built-in not_null, unique, accepted_values, relationships; custom SQL tests | Model-level tests |
| **Elementary** | dbt package for anomaly detection + data observability lineage | dbt-native observability |
| **DQOps** | Open-source data quality platform with profiling and automation | Full lifecycle data quality |
| **Metaplane** | Data observability monitoring for warehouses | Anomaly detection SaaS |
| **DataKitchen** | End-to-end data journey observability and testing | Automated test generation |
| **Grai** | Data catalog + CI integration for downstream impact testing | Pipeline impact analysis |
| **Provero** | Declarative data quality engine with YAML checks | Vendor-neutral contracts |

**Great Expectations example:**
```python
import great_expectations as gx

context = gx.get_context()
validator = context.sources.pandas_default.read_csv("orders.csv")

# Define expectations
validator.expect_column_to_exist("order_id")
validator.expect_column_values_to_not_be_null("order_id")
validator.expect_column_values_to_be_unique("order_id")
validator.expect_column_values_to_be_between(
    column="revenue", min_value=0, max_value=100000
)
validator.expect_column_values_to_be_in_set(
    column="status",
    value_set=["pending", "shipped", "delivered", "cancelled"]
)

# Save and run validation
validator.save_expectation_suite()
results = validator.validate()
print(f"Success: {results.success}")
```

---

## 18. Data Governance & Catalog

Discoverability, lineage, and compliance for your data assets.

| Tool | Description | Best For |
|------|-------------|----------|
| **Apache Atlas** | Data governance and metadata framework (Hadoop ecosystem) | On-prem Hadoop governance |
| **DataHub (LinkedIn)** | Generalized metadata search and discovery | Enterprise metadata catalog |
| **Amundsen (Lyft)** | Metadata catalogue with data discovery | Self-service data discovery |
| **OpenMetadata** | All-in-one: lineage, catalog, quality, governance | Open-source unified platform |
| **OpenLineage** | Open standard for lineage metadata (event-based) | Cross-tool lineage federation |
| **Apache Gravitino** | Unified metadata management across engines | Multi-engine catalog |

**Why lineage matters:**
- If `dim_customers` breaks, which dashboards are affected?
- When you change a column, which downstream models depend on it?
- DataHub + OpenLineage together answer both questions automatically

---

## 19. Business Intelligence & Visualization

Turning data into decisions.

| Tool | Description | User Type | Best For |
|------|-------------|-----------|----------|
| **Apache Superset** | Enterprise-ready open-source BI with SQL Lab | Technical + Business | Self-hosted BI at any scale |
| **Metabase** | Easiest open-source BI — non-technical users can explore | Business users | Self-service analytics |
| **Redash** | Connect any data source, SQL-first dashboard | Technical analysts | SQL-first dashboard building |
| **Tableau** | Industry leader with powerful drag-and-drop | Business users | Enterprise visual analytics |
| **Power BI** | Microsoft's BI integrated with Office 365 | Business users | Microsoft-ecosystem shops |
| **Looker / Looker Studio** | Google's semantic-layer-driven BI | Technical analysts | LookML model-driven analytics |
| **Hex** | Collaborative notebook + app builder | Data scientists | Sharing analysis as interactive apps |
| **Evidence** | Build data apps from SQL and Markdown | Engineers | Code-first data apps |
| **Lightdash** | Open-source Looker alternative using dbt models | Technical analysts | dbt-first BI |
| **D3.js** | Low-level JavaScript data visualization library | Engineers | Custom interactive visualizations |
| **Plotly / Dash** | Python interactive dashboards | Python developers | Python-native dashboards |
| **Apache Superset + dbt** | Best open-source BI + transformation combo | Mixed | Full open-source analytics stack |

---

## 20. Messaging Infrastructure

The plumbing for event-driven architectures.

| Tool | Description | Protocol | Best For |
|------|-------------|----------|----------|
| **Apache Kafka** | Distributed commit log — millions of messages/sec | Custom (TCP) | Event sourcing, CDC, data pipelines |
| **Apache Pulsar** | Multi-tenant distributed pub-sub with native Geo-replication | Custom | Multi-cluster global messaging |
| **RabbitMQ** | Classic AMQP broker with flexible routing | AMQP | Task queues, work distribution |
| **Apache ActiveMQ** | Flexible multi-protocol messaging | AMQP/STOMP/MQTT | Enterprise JMS/AMQP workloads |
| **NATS** | Simple, secure, high-performance messaging | NATS | Cloud-native microservices |
| **AWS Kinesis** | Managed Kafka-like streaming on AWS | HTTP/REST | AWS-native streaming |
| **Azure Event Hubs** | Managed event streaming on Azure | AMQP/Kafka-compatible | Azure-native + Kafka migration |
| **Google Pub/Sub** | Managed messaging on GCP | HTTP/gRPC | GCP-native async messaging |
| **ZeroMQ** | Lightweight socket-level messaging library | Sockets | High-performance in-process messaging |
| **Nakadi (Zalando)** | RESTful event bus over Kafka | HTTP/REST | HTTP-first Kafka publishing |

**Kafka vs RabbitMQ:**
```
Kafka:
  ✓ Log retention (replay events days/weeks later)
  ✓ High throughput (millions of msgs/sec)
  ✓ Consumer groups scale horizontally
  ✗ No per-message acknowledgement routing

RabbitMQ:
  ✓ Flexible routing (direct, fanout, topic exchanges)
  ✓ Per-message TTL and dead letter queues
  ✓ Traditional task queue semantics
  ✗ Messages deleted after consumption (no replay)
```

---

## 21. Monitoring & Observability

Knowing when your pipelines break before users notice.

| Tool | Description | Best For |
|------|-------------|----------|
| **Prometheus** | Open-source metrics collection + alerting | Infrastructure metrics, SLO tracking |
| **Grafana** | Open-source analytics and monitoring dashboards | Visualizing Prometheus/Loki/Tempo metrics |
| **Elasticsearch + Kibana** | ELK stack for log search and dashboards | Log analytics, APM |
| **Logstash** | Server-side log processing pipeline (E**L**K) | Log ingestion and enrichment |
| **DataDog** | Full-stack monitoring SaaS (metrics + logs + APM + synthetics) | Enterprise end-to-end observability |
| **Grafana Loki** | Log aggregation system by Grafana Labs | Kubernetes log aggregation |
| **OpenTelemetry** | Vendor-neutral observability framework (traces + metrics + logs) | Instrumentation standardization |
| **cAdvisor** | Container resource usage and performance analysis | Docker/Kubernetes container metrics |

**Prometheus alerting rule for pipeline lag:**
```yaml
# alert if Kafka consumer lag > 100k messages for 5 minutes
groups:
  - name: kafka_alerts
    rules:
      - alert: HighKafkaConsumerLag
        expr: kafka_consumer_lag_sum > 100000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Kafka consumer lag is high"
          description: "Consumer group {{ $labels.consumer_group }} lag: {{ $value }}"
```

---

## 22. Data Comparison & Profiling

Understanding and validating your data's quality and structure.

| Tool | Description | Best For |
|------|-------------|----------|
| **datacompy** | Compare two DataFrames (Pandas, Polars, Spark) with detailed diff | Migration validation, DE testing |
| **YData Profiling** | One-line data profile with statistics, correlations, missing values | Exploratory data analysis |
| **DataProfiler (CapOne)** | Python library for profiling + PII/sensitive data detection | Data quality + compliance |
| **DVT (Google)** | Data Validation Tool for source-to-target comparison | Migration data reconciliation |
| **Desbordante** | Advanced pattern discovery (FDs, UCC, MDs) in large datasets | Data contract verification |

```python
# YData Profiling - one-liner EDA report
import pandas as pd
from ydata_profiling import ProfileReport

df = pd.read_parquet("orders.parquet")
profile = ProfileReport(df, title="Orders Data Profile")
profile.to_file("orders_profile.html")
# Opens a full HTML report with: statistics, distributions,
# correlations, missing values, duplicates, interactions
```

---

## 23. Infrastructure & Containerization

The platform your data tools run on.

| Tool | Description | Best For |
|------|-------------|----------|
| **Docker** | Container runtime — package app + dependencies | Reproducible environments |
| **Kubernetes** | Container orchestration at scale | Production container management |
| **Terraform** | Infrastructure as Code for any cloud | Reproducible cloud infrastructure |
| **Helm** | Kubernetes package manager | Deploying Airflow/Kafka/Spark on K8s |
| **Rancher** | Kubernetes management platform | Multi-cluster K8s management |
| **Nomad** | Lightweight cluster manager (simpler than K8s) | Smaller-scale workload scheduling |
| **cAdvisor** | Container resource monitoring | Container performance visibility |

**Terraform for data infrastructure:**
```hcl
# Production data stack on AWS
module "data_lake" {
  source = "./modules/s3"
  bucket_name     = "prod-data-lake"
  lifecycle_rules = true
  versioning      = true
}

module "warehouse" {
  source        = "./modules/redshift"
  cluster_type  = "multi-node"
  node_type     = "ra3.4xlarge"
  num_nodes     = 3
  database_name = "analytics"
}

module "orchestration" {
  source       = "./modules/mwaa"  # Managed Airflow
  environment  = "prod"
  dag_s3_path  = "dags/"
}
```

---

## 24. Public Datasets

Free data for learning, prototyping, and portfolio projects.

| Dataset | Description | Size | Use Case |
|---------|-------------|------|----------|
| **NYC Yellow Taxi Trips** | Hundreds of millions of taxi trips with pickup/dropoff, fare, tip | ~10GB/year | Batch processing, Spark, dbt |
| **GitHub Archive** | Every public GitHub event since 2011 (hourly updates) | 5TB+ | Streaming + time series analysis |
| **Common Crawl** | Monthly web crawl of billions of pages | Petabytes | NLP, ML training data |
| **Wikipedia Dumps** | Complete text of all Wikipedia articles | ~20GB | NLP, graph analysis |
| **Reddit Comments** | Public comments and submissions | TBs | NLP, sentiment analysis |
| **DexPaprika** | Real-time DEX price data (34 blockchains, 30M+ pools) | Streaming | Real-time pipeline development |
| **Eventsim** | Synthetic event data simulator (music streaming) | Configurable | Streaming pipeline testing |
| **Twitter/X Streaming** | Real-time tweet stream | Streaming | Social media analytics |

**Recommended project datasets by skill level:**
```
Beginner:   NYC Taxi + PostgreSQL + dbt
Intermediate: NYC Taxi + Spark + BigQuery + Looker
Advanced:   GitHub Archive + Kafka + Flink + Delta Lake + Superset
```

---

## 25. Company Tool Ecosystem Map

Where major companies invest and what they've built.

### By Category

**Orchestration**
| Company | Tool |
|---------|------|
| Astronomer | Managed Airflow + Astro SDK |
| Prefect | Prefect Cloud + Server |
| Dagster Labs | Dagster + Dagster Cloud |
| Kestra | Kestra (YAML-first) |
| Shipyard | Managed Airflow alternative |
| Mage | Mage.ai (visual + code) |
| Apache | Airflow (open-source) |

**Data Lake / Lakehouse**
| Company | Format/Platform |
|---------|----------------|
| Databricks | Delta Lake + Unity Catalog |
| Tabular (acquired by Databricks) | Managed Apache Iceberg |
| Onehouse | Managed Apache Hudi |
| Apache | Iceberg, Hudi, Spark |
| Microsoft | Microsoft Fabric + ADLS |
| Apple | Apache Iceberg (Quartz) |

**Data Warehouse**
| Company | Product |
|---------|---------|
| Snowflake | Snowflake (cloud DW) |
| Google | BigQuery |
| Amazon | Redshift |
| Microsoft | Azure Synapse |
| Firebolt | Firebolt (sub-second analytics) |

**Data Quality**
| Company | Product |
|---------|---------|
| dbt Labs | dbt tests + Elementary |
| Great Expectations | GX Cloud |
| Soda | Soda Core + Soda Cloud |
| Metaplane | Metaplane (observability) |
| Gable | Data contracts platform |
| DQOps | DQOps (open-source) |

**Data Integration / ELT**
| Company | Product |
|---------|---------|
| Fivetran | Managed ELT (300+ connectors) |
| Airbyte | Open-source ELT |
| dltHub | dlt (Python library) |
| Meltano | CLI ELT (Singer) |
| Estuary | Flow (real-time ELT) |
| Sling | Sling CLI |

**Analytics / BI**
| Company | Product |
|---------|---------|
| Tableau (Salesforce) | Tableau Desktop/Server |
| Microsoft | Power BI |
| Google | Looker / Looker Studio |
| Preset | Managed Apache Superset |
| Starburst | Managed Trino |
| Hex | Collaborative notebooks |
| Evidence | Code-first data apps |

**Modern OLAP**
| Company | Product |
|---------|---------|
| Apache | Druid, Pinot, Kylin |
| ClickHouse Inc | ClickHouse Cloud |
| DuckDB Labs | DuckDB |
| QuestDB | QuestDB Cloud |
| StarRocks | StarRocks |

### Tech Stack Blueprints

**Startup Stack (low cost, managed)**
```
Ingest:      dlt or Airbyte (open-source)
Storage:     AWS S3 (data lake) + PostgreSQL (operational)
Warehouse:   BigQuery or Snowflake
Transform:   dbt Core
Orchestrate: Prefect or Kestra
BI:          Metabase or Redash
Quality:     dbt tests + Elementary
Cost:        ~$200-500/month
```

**Scale-up Stack (millions of events/day)**
```
Ingest:      Kafka + Debezium CDC + Fivetran
Storage:     AWS S3 + Delta Lake
Warehouse:   Redshift or Snowflake
Transform:   dbt Cloud + Spark
Orchestrate: Airflow (MWAA) or Dagster
BI:          Tableau or Looker
Quality:     Great Expectations + Metaplane
Lineage:     DataHub + OpenLineage
Cost:        ~$5,000-20,000/month
```

**Enterprise / Big Tech Stack**
```
Ingest:      Kafka (multi-region) + custom connectors + Debezium
Storage:     Multi-cloud S3 + Iceberg tables + Alluxio caching
Warehouse:   BigQuery + Redshift (polyglot)
Transform:   Spark on EMR/Dataproc + dbt
Orchestrate: Airflow + internal scheduler
BI:          Tableau + Looker + internal dashboards
Quality:     Custom data contracts + Great Expectations + on-call alerts
Infra:       Kubernetes + Terraform + GitHub Actions CI/CD
Cost:        Millions/year
```

---

## Quick Tool Picker Reference

```
Need to...                                    → Use
─────────────────────────────────────────────────────────────────────
Ingest from SaaS tools (Salesforce, HubSpot)  → Fivetran or Airbyte
Stream events in real-time                     → Apache Kafka
Process streaming data                         → Apache Flink or Spark Streaming
Store structured analytical data               → BigQuery or Snowflake
Build a data lake                              → S3 + Delta Lake or Iceberg
Transform SQL data                             → dbt
Schedule and monitor pipelines                 → Airflow or Dagster
Validate data quality                          → Great Expectations or dbt tests
Visualize data for business                    → Tableau, Metabase, or Superset
Discover data catalog/lineage                  → DataHub or OpenMetadata
Monitor infrastructure                         → Prometheus + Grafana
Manage infrastructure as code                  → Terraform
Containerize data tools                        → Docker + Kubernetes
Do time-series analytics                       → TimescaleDB or InfluxDB
Run in-process analytics on laptop             → DuckDB
Do graph analytics                             → Neo4j
Cache data lake reads                          → Alluxio
Version your data lake                         → lakeFS
Capture CDC from Postgres                      → Debezium
Analyze data locally without spinning up infra → DuckDB + Parquet files
```

---

← [09 Hands-On Course](09-hands-on-course.md) | [11 Case Studies →](11-case-studies.md)
