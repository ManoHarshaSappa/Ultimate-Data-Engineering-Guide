# 02 — Data Engineering Fundamentals

> **Guide by [Mano Harsha Sappa](https://www.linkedin.com/in/manoharshasappa/)** | [Portfolio](https://manoharshasappa.github.io/portfolio_ManoHarshaSappa/) | [Back to Index](../README.md)

---

## Contents

- [The 4 Vs of Big Data](#the-4-vs-of-big-data)
- [Why Big Data?](#why-big-data)
- [ETL vs ELT](#etl-vs-elt)
- [Batch vs Stream Processing](#batch-vs-stream-processing)
- [Lambda vs Kappa Architecture](#lambda-vs-kappa-architecture)
- [Data Warehouse vs Data Lake vs Lakehouse](#data-warehouse-vs-data-lake-vs-lakehouse)
- [Data Platform Architecture](#data-platform-architecture)
- [81 Platform Design Questions](#81-platform-design-questions)
- [Scaling: Up vs Out](#scaling-up-vs-out)

---

## The 4 Vs of Big Data

Big Data is defined by four characteristics. Volume is only one part of it:

### Volume — How Much Data You Have
The size of your dataset. Can be gigabytes, terabytes, or petabytes.

- Small company: GBs of data → SQL database is fine
- Medium company: TBs of data → Data warehouse + basic pipelines
- Large company: PBs of data → Distributed systems (Hadoop, Spark)

### Velocity — How Fast Data Is Coming In
The speed at which data arrives. This drives the need for streaming systems.

- Slow: Daily batch files → Airflow + Spark batch jobs
- Medium: Hourly feeds → Scheduled pipelines
- Fast: Real-time events → Kafka + Flink/Spark Streaming

**Example**: A payment processor handles thousands of transactions per second. That's high velocity — you can't wait until end of day to detect fraud. You need stream processing.

### Variety — How Different Your Data Is
The diversity of data types and formats from different sources.

- Structured: CSV, SQL tables with fixed schemas
- Semi-structured: JSON, XML, Avro — schema is embedded in the data
- Unstructured: Text, images, audio, video

**The engineering challenge**: Joining sales data (from SQL), web logs (JSON), user profiles (CSV), and sensor readings (binary) into one coherent view.

### Veracity — How Reliable Your Data Is
The trustworthiness and accuracy of data. Big data is often messy.

- IoT sensors fail and send null readings
- Humans make typos in form fields
- Systems go down and lose data
- Third-party APIs return inconsistent formats

**As a data engineer**, you must handle: missing values, duplicates, schema changes, late-arriving data, and corrupt records.

---

## Why Big Data?

The core business problem is **catastrophic success** — exponential growth that your data infrastructure isn't designed to handle.

```
Day 1:     1 user → Simple SQL DB, works great
Month 3:   1,000 users → Still fine
Month 6:   100,000 users → Getting slow
Month 12:  10,000,000 users → Database melts down
Month 18:  You fail. Not because the product failed — because the data infrastructure couldn't scale.
```

The hockey stick growth curve:
```
1 → 2 → 4 → 8 → 16 → 32 → 64 → 128 → 256 → 512 → 1,024 → BOOM
```

**Planning is everything.** If you design for big data from day one (even if you don't need it yet), you protect yourself from this failure mode. The cost of premature optimization is much lower than the cost of emergency scaling.

---

## ETL vs ELT

These are the two paradigms for moving data from sources into your analytical store.

### Classic ETL (Extract → Transform → Load)

```
Source DB → [Extract raw data] → [Transform on separate server] → [Load into warehouse]
```

- Transform happens BEFORE loading
- Heavy compute required in the "middle" layer
- Common tools: Apache Spark, custom Python, SSIS, Informatica
- Works well when the warehouse is expensive to run (old on-prem Teradata)

**Problem**: The ETL process becomes a bottleneck as data grows. Transformations on large datasets take hours. The middle layer is a single point of failure.

### Modern ELT (Extract → Load → Transform)

```
Source DB → [Extract raw data] → [Load raw into warehouse] → [Transform inside warehouse]
```

- Raw data lands in the warehouse first (Bronze layer)
- Transformations happen INSIDE the warehouse using SQL
- Common tools: dbt, Fivetran, Airbyte, dlt
- Works great with cloud warehouses (Snowflake, BigQuery, Redshift) which have massive compute

**Advantages of ELT**:
1. **Speed**: Load first, ask questions later
2. **Flexibility**: Raw data preserved — you can re-transform anytime
3. **Scalability**: Cloud warehouses scale compute elastically
4. **Simplicity**: dbt makes transformations version-controlled SQL

**Is ETL Dead?** No — it still makes sense when:
- You have strict data privacy requirements (must mask PII before loading)
- Warehouse compute is expensive and you want to pre-filter data
- You're moving data between systems with very different schemas

---

## Batch vs Stream Processing

### Batch Processing

Process data in large chunks at scheduled intervals.

```
Data accumulates → Scheduled job triggers → Process all data → Output results
```

- **Analogy**: Doing your taxes once a year — you collect all receipts, then process them at once
- **Latency**: Minutes to hours
- **Use cases**: Daily reports, monthly billing, historical analysis, ML model training
- **Tools**: Apache Spark, Hive, MapReduce

**The classic batch pipeline**:
```
Data Source → Storage (S3/HDFS) → Batch job (Spark) → Data Warehouse → Dashboard
```

**When to use batch**:
- Data doesn't need to be fresh immediately
- Large historical datasets
- Complex transformations that take time
- Lower cost (run once, not continuously)

### Stream Processing

Process data in real-time as it arrives, event by event.

```
Event arrives → Immediately processed → Result available in milliseconds/seconds
```

- **Analogy**: A bank monitoring every credit card transaction instantly for fraud
- **Latency**: Milliseconds to seconds
- **Use cases**: Fraud detection, real-time dashboards, recommendation engines, IoT monitoring
- **Tools**: Apache Kafka, Apache Flink, Spark Streaming, AWS Kinesis

**The streaming pipeline**:
```
Events → Kafka → Flink/Spark Streaming → Real-time DB/API → Live Dashboard
```

**When to use streaming**:
- You need to act on data within seconds
- Triggering alerts based on threshold breaches
- Real-time personalization
- Monitoring systems (infrastructure, application performance)

### Three Methods of Message Delivery in Streaming

**At Least Once**
- A message is processed one OR MORE times
- Some duplication is acceptable
- Example: GPS tracking — if a car's location is processed twice with the same timestamp, no harm done
- Most common default

**At Most Once**
- A message is processed ZERO OR ONE time
- Dropping messages is acceptable
- Example: Log aggregation for approximate metrics
- Simplest to implement

**Exactly Once**
- A message is processed EXACTLY ONE time — no duplicates, no drops
- Hardest to achieve, requires distributed transactions
- Example: Banking transactions — a payment must be processed exactly once
- Tools: Kafka with transactions, Apache Flink with checkpoints

**Rule of thumb**: Start with "at least once" and add deduplication logic. Use "exactly once" only when business logic demands it.

### Should You Do Batch or Stream?

**Start with batch.** It's simpler, cheaper, and solves 80% of use cases. Add streaming only when:
- You have a specific business need for real-time (not just "real-time sounds cool")
- The latency of batch is causing actual business problems
- You have the engineering capacity to maintain streaming infrastructure

---

## Lambda vs Kappa Architecture

### Lambda Architecture

Two parallel layers: batch for accuracy, streaming for speed.

```
Data → [Batch Layer (Hadoop/Spark)] ──→ ┐
     → [Speed Layer (Kafka/Flink)]  ──→ → Serving Layer → Queries
```

- **Batch layer**: Reprocesses all historical data periodically for accuracy
- **Speed layer**: Processes recent data in real-time for low latency
- **Serving layer**: Merges results from both layers

**Problems with Lambda**:
- Maintaining two codebases (batch + streaming) for the same logic
- Keeping them consistent is hard
- High operational complexity

### Kappa Architecture

Only one layer: everything is streaming.

```
Data → [Streaming Layer (Kafka + Flink)] → Serving Layer → Queries
```

- For historical reprocessing, replay from Kafka (extend retention)
- Single codebase, simpler operations
- Modern approach used by LinkedIn, Netflix, Uber

**When to use which**:
- **Lambda**: When you have strict accuracy requirements AND can't lose historical data
- **Kappa**: When your streaming system can replay data and handle backfill (preferred modern approach)

---

## Data Warehouse vs Data Lake vs Lakehouse

### Data Warehouse

A structured, optimized storage system for analytical queries.

```
Characteristics:
✓ Structured data only (tables with schemas)
✓ Highly optimized for SQL analytics
✓ ETL/ELT pipelines clean data before loading
✓ Expensive but fast
✗ No unstructured data
✗ Rigid schema changes
```

**Examples**: Snowflake, Google BigQuery, Amazon Redshift, Azure Synapse

**Best for**: Business intelligence, structured reporting, regulatory compliance

### Data Lake

A flat storage of raw data in native format.

```
Characteristics:
✓ Stores everything: structured, semi-structured, unstructured
✓ Cheap storage (S3, GCS, ADLS)
✓ Schema-on-read (define schema when querying)
✓ Flexible for ML and data science
✗ Can become a "data swamp" without governance
✗ Slower queries than warehouse
✗ No ACID transactions (traditionally)
```

**Examples**: AWS S3 + Athena, Azure Data Lake Storage, Google Cloud Storage

**Best for**: Raw data storage, ML feature stores, archival

### Data Lakehouse

The best of both worlds: data lake storage + warehouse-like performance and governance.

```
Characteristics:
✓ Open file formats (Parquet, ORC)
✓ ACID transactions
✓ Schema enforcement + evolution
✓ Time travel (query historical versions of data)
✓ Upserts and deletes on lake data
✓ Cheaper than pure warehouse
```

**Technologies**: Delta Lake (Databricks), Apache Iceberg, Apache Hudi

**Best for**: Modern data platforms that need flexibility + performance + governance

### Comparison Table

| Feature | Data Warehouse | Data Lake | Data Lakehouse |
|---------|---------------|-----------|----------------|
| Storage Cost | High | Low | Low |
| Query Performance | Excellent | Variable | Good-Excellent |
| Data Types | Structured | All types | All types |
| ACID Transactions | Yes | No (traditional) | Yes |
| Schema | On-write | On-read | Both |
| Time Travel | Limited | No | Yes |
| ML Support | Limited | Excellent | Excellent |
| Examples | Snowflake, BigQuery | S3+Athena | Delta Lake, Iceberg |

---

## Data Platform Architecture

### The Full Platform Reference Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    DATA SOURCES                                  │
│  Databases │ APIs │ Files │ IoT Sensors │ SaaS Apps │ Streams   │
└──────────────────────────┬──────────────────────────────────────┘
                           │ Ingest
┌──────────────────────────▼──────────────────────────────────────┐
│                   INGESTION / CONNECT LAYER                      │
│  Airbyte │ Fivetran │ dlt │ Apache NiFi │ Kafka Connect         │
│  Logstash │ FluentD │ Apache Sqoop │ Debezium (CDC)             │
└──────────────────────────┬──────────────────────────────────────┘
                           │
        ┌──────────────────┼──────────────────┐
        │ Batch            │ Stream            │
┌───────▼────────┐  ┌─────▼──────────────────▼──────────────┐
│ Raw Storage    │  │ Message Buffer                          │
│ S3 / GCS /    │  │ Apache Kafka / AWS Kinesis /            │
│ ADLS           │  │ Google Pub/Sub / RabbitMQ               │
└───────┬────────┘  └────────────────────┬───────────────────┘
        │                                │
┌───────▼────────────────────────────────▼───────────────────┐
│                   PROCESSING LAYER                           │
│  Batch: Apache Spark │ Hive │ MapReduce │ dbt               │
│  Stream: Apache Flink │ Spark Streaming │ KSQL              │
└────────────────────────────┬───────────────────────────────┘
                             │
┌────────────────────────────▼───────────────────────────────┐
│                    STORAGE LAYER                            │
│  Analytical: Snowflake │ BigQuery │ Redshift │ Databricks   │
│  Lakehouse:  Delta Lake │ Apache Iceberg │ Apache Hudi      │
│  OLTP:       PostgreSQL │ MySQL │ MongoDB                   │
│  Cache:      Redis │ Memcached                              │
│  Search:     Elasticsearch                                  │
└────────────────────────────┬───────────────────────────────┘
                             │
┌────────────────────────────▼───────────────────────────────┐
│                  ORCHESTRATION LAYER                         │
│         Apache Airflow │ Prefect │ Kestra │ Dagster          │
└────────────────────────────┬───────────────────────────────┘
                             │
┌────────────────────────────▼───────────────────────────────┐
│                 CONSUMPTION / VISUALIZE LAYER               │
│  BI Tools: Tableau │ Power BI │ Looker │ Superset           │
│  APIs:     FastAPI │ GraphQL │ REST                         │
│  ML:       Feature stores │ Model serving │ MLflow          │
└────────────────────────────────────────────────────────────┘
```

### The Medallion Architecture (Bronze → Silver → Gold)

Modern data platforms organize data into quality layers:

**Bronze (Raw)**
- Exact copy of source data, no transformation
- Append-only, immutable
- Stored in native format (JSON, CSV, Avro)
- Purpose: Data lineage, debugging, re-processing

**Silver (Cleansed)**
- Cleaned, deduplicated, validated
- Consistent schemas
- Joined across sources
- Purpose: Data scientists, analysts, reporting

**Gold (Curated)**
- Business-level aggregations
- Optimized for specific use cases
- Often denormalized for query performance
- Purpose: Dashboards, APIs, executive reporting

```
Raw API data → [Bronze: S3/Parquet] → [Silver: cleaned, joined] → [Gold: aggregated metrics]
```

---

## 81 Platform Design Questions

Before building any data platform, answer these questions. They guide every architectural decision.

### Data Source Questions

**Origin & Structure**
- What is the data source? (Database, API, file, IoT device)
- What format does data arrive in? (JSON, CSV, Avro, Parquet, binary)
- What is the schema? Is it fixed or dynamic?
- Is the data structured, semi-structured, or unstructured?
- How are schema changes handled? (schema evolution strategy)

**Volume & Velocity**
- How much data per transmission?
- How fast is data arriving? (events/minute)
- Maximum data volume expected per day?
- Expected growth over the next 1, 2, 5 years?
- Are there peak periods? (Black Friday, New Year's Eve)
- How will the system handle burst traffic?

**Source Reliability**
- Do messages arrive late? If so, how late?
- Is there a risk of duplicate data?
- How reliable are sources? What's the expected failure rate?
- What happens if a source goes offline?
- Do we need retry logic for failed transmissions?

**Security & Compliance**
- Does data need encryption at rest and in transit?
- What compliance frameworks apply? (GDPR, HIPAA, SOX, PCI)
- Is there a requirement for data masking or anonymization?
- Who owns the data? What are the access controls?

**Metadata & Auditing**
- What metadata should be captured with each record?
- How do we track data lineage?
- What audit logging is required?

### Destination & Use Case Questions

**Use Case**
- What kind of use case is this? (Analytics, BI, ML, operational, visualization)
- What downstream systems will consume this data?
- How critical is real-time vs historical data?

**Query & Performance Requirements**
- How is data visualized? Raw or aggregated?
- How fast do results need to appear? (SLA)
- How much data will be queried at once?
- How often is data queried?
- What are concurrent user requirements?
- What is the acceptable query response time?

**Data Freshness**
- How fresh does data need to be?
  - Real-time (< 1 second)?
  - Near real-time (seconds)?
  - Hourly?
  - Daily?

**Lifecycle & Retention**
- What's the data retention period?
- What's the archival strategy?
- Will old data need re-processing?
- Hot vs. cold storage strategy?

**Access & Security**
- Who has access to query the data?
- What tools do they use to query?
- Is role-based access control (RBAC) needed?
- How is sensitive data protected? (row-level security)

**Scaling**
- What are scalability requirements as data grows?
- How will the system handle 10x the current load?
- What is the disaster recovery strategy?

---

## Scaling: Up vs Out

### Scaling Up (Vertical Scaling)

Buy a bigger machine: more CPU, more RAM, faster disk.

```
Before: 4 cores, 16GB RAM, 500GB SSD
After:  32 cores, 128GB RAM, 4TB SSD
```

**Pros**: Simple, no code changes needed  
**Cons**: Exponentially expensive, has a physical limit, single point of failure  
**Best for**: Small-medium workloads, quick fix for immediate problems

### Scaling Out (Horizontal Scaling)

Add more machines and distribute the load.

```
Before: 1 server handling all queries
After:  10 servers each handling 1/10 of queries
```

**Pros**: Theoretically unlimited scale, fault tolerant (one node fails, others continue)  
**Cons**: Complex distributed system, data needs to be partitioned, network overhead

**How data systems scale out**:
- **HDFS**: Adds more data nodes, files split into blocks across nodes
- **Kafka**: Adds more brokers, topics split into more partitions
- **Spark**: Adds more executor nodes to cluster
- **Snowflake**: Adds more virtual warehouses (compute clusters)

### When to Use Big Data Tools

**Use big data tools ONLY when you actually need them.**

Big data infrastructure (Hadoop clusters, Spark clusters, Kafka) is expensive to build, maintain, and operate. A Hadoop cluster needs minimum 5 nodes to function properly.

**You probably DON'T need big data tools if**:
- Your data fits on a single machine
- Your biggest dataset is < 100GB
- You can query your data in < 1 hour on a modern server

**You DO need big data tools when**:
- Single-machine processing would take days
- You need fault tolerance (you can't afford to lose a job)
- You need to process data faster than one machine can handle
- Your data grows by gigabytes per day and won't stop

---

## Key Architecture Patterns

### Pattern 1: Simple Batch (Good for most companies)
```
Source DB → Python ETL → Snowflake → Tableau
```

### Pattern 2: Orchestrated ELT (Modern standard)
```
Sources → Airbyte → Raw S3/BigQuery → dbt → Gold tables → BI
                                        ↑
                                    Airflow DAG
```

### Pattern 3: Lambda Architecture (High-accuracy streaming)
```
Sources → Kafka → [Flink streaming] → Real-time Redis
                → [Spark batch]    → Data Warehouse
```

### Pattern 4: Full Lakehouse (Enterprise)
```
All Sources → dlt/Fivetran → Bronze S3+Iceberg → Silver → Gold
                          ↑                    ↑
                     Spark Streaming        dbt + Airflow
                                              ↓
                                    Snowflake/BigQuery serving layer
```

---

*← [01 - Introduction](01-introduction.md) | [03 - Essential Skills](03-essential-skills.md) →*
