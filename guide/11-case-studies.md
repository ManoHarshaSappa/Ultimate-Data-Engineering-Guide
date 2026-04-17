# Case Studies: How Top Companies Do Data Engineering

**Author:** Mano Harsha Sappa | [LinkedIn](https://www.linkedin.com/in/manoharshasappa/) | [Portfolio](https://manoharshasappa.github.io/portfolio_ManoHarshaSappa/) | [Email](mailto:sappamanoharsha@gmail.com)

---

## Contents

1. [Netflix — The Keystone Platform](#1-netflix--the-keystone-platform)
2. [Uber — Big Data Platform & AresDB](#2-uber--big-data-platform--aresdb)
3. [LinkedIn — Kafka Origin & Samza](#3-linkedin--kafka-origin--samza)
4. [Spotify — Event Delivery to the Cloud](#4-spotify--event-delivery-to-the-cloud)
5. [Airbnb — Druid & Spark Streaming](#5-airbnb--druid--spark-streaming)
6. [Pinterest — MySQL to S3 via Kafka](#6-pinterest--mysql-to-s3-via-kafka)
7. [Twitter — Heron Stream Processing](#7-twitter--heron-stream-processing)
8. [Zalando — Delta Lake at Scale](#8-zalando--delta-lake-at-scale)
9. [CERN — Apache Spark for Physics](#9-cern--apache-spark-for-physics)
10. [Facebook/Meta — Spark at 60TB](#10-facebookmeta--spark-at-60tb)
11. [Booking.com — Kafka Ecosystem Management](#11-bookingcom--kafka-ecosystem-management)
12. [Lyft — Running Airflow at Scale](#12-lyft--running-airflow-at-scale)
13. [ING Bank — Streaming Fraud Detection](#13-ing-bank--streaming-fraud-detection)
14. [Instagram — 300TB Spark Pipelines](#14-instagram--300tb-spark-pipelines)
15. [Dropbox — Finding Kafka's Throughput Limit](#15-dropbox--finding-kafkas-throughput-limit)
16. [eBay — Spark as Core ETL Platform](#16-ebay--spark-as-core-etl-platform)
17. [Disney — Auto-Scaling Kinesis Streams](#17-disney--auto-scaling-kinesis-streams)
18. [Expedia — Spark Streaming + Kafka Best Practices](#18-expedia--spark-streaming--kafka-best-practices)
19. [NASA — Apache Science Data Analytics Platform](#19-nasa--apache-science-data-analytics-platform)
20. [Woot/Amazon — Serverless Data Lake on AWS](#20-wootamazon--serverless-data-lake-on-aws)
21. [Slack — Streaming Data Pipelines](#21-slack--streaming-data-pipelines)
22. [Drivetribe — Kappa Architecture with Flink](#22-drivetribe--kappa-architecture-with-flink)
23. [Tinder — Scalable Monitoring with Spark](#23-tinder--scalable-monitoring-with-spark)
24. [Grammarly — Versatile Analytics on Spark](#24-grammarly--versatile-analytics-on-spark)
25. [PayPal — Scalable Event Pipeline](#25-paypal--scalable-event-pipeline)
26. [Key Lessons Across All Case Studies](#26-key-lessons-across-all-case-studies)
27. [Architecture Patterns Summary](#27-architecture-patterns-summary)

---

## Why Case Studies Matter

Reading about tools is one thing. Understanding **why** a company chose Kafka over RabbitMQ, or why they moved from MapReduce to Spark, is how you build engineering judgment. Every case study here carries a decision — a constraint, a scale problem, a cost problem — that led to a specific architectural choice.

Study these not to copy, but to build a library of patterns you can draw from when you face similar constraints.

---

## 1. Netflix — The Keystone Platform

**Scale:** 75M+ users, 125M hours of content watched daily, **500 billion user events per day**, 1.3 petabytes/day

**Business problem:** Keep subscribers watching. Recommend the right content. Know what show gets you hooked, by country.

### The Evolution

**Phase 1: Batch (Chukwa Pipeline)**
```
User Action → Chukwa (log collector) → Hadoop Sequence Files on S3
                                      → Elastic MapReduce (daily/hourly jobs)
                                      → Recommendations (1 hour to 1 day stale)
```
Chukwa collected logs into Hadoop sequence files stored on Amazon S3. EMR ran MapReduce jobs hourly and daily. This worked at first — but looking into the past wasn't enough.

**Phase 2: Real-Time (Keystone Pipeline)**
```
Client (browser/mobile)
    ↓ HTTP
"Viewing History" service  ←→  Play events (what you watch, where you pause)
"Beacon" service           ←→  Impression events (scroll, click, browse)
    ↓
Apache Kafka (buffer between services and analytics)
    ↓
Spark Streaming (analytics — exact algorithms are trade secret)
    ↓
Cassandra KV store (trending data)
    ↓
"Recommender Service" → Client
```

**Key technologies:**
- Apache Kafka: ingestion buffer
- Apache Spark Streaming: real-time analytics
- Cassandra: low-latency KV store for trending data
- Amazon S3: data lake for batch processing
- Amazon EMR: managed MapReduce/Spark

**The "Trending Now" feature** uses two event streams:
1. **Play events** — what you watched, where you stopped, 30s rewind usage
2. **Impression events** — scrolling, clicking in the Netflix library

Combined, these tell Netflix not just what's watched, but what people are **discovering**.

**Lesson:** Batch processing tells you what happened. Real-time streaming tells you what's happening NOW. Netflix built both and used each for the right use case.

**Netflix engineering blog:** [netflixtechblog.com](https://netflixtechblog.com/tagged/big-data)

---

## 2. Uber — Big Data Platform & AresDB

**Scale:** Millions of trips/day, complex surge pricing, driver allocation, real-time ETA

**Business problem:** Surge pricing must be computed in real-time. Driver matching cannot wait for batch jobs. A stale price shown to a customer is a broken promise.

### Uber's Big Data Platform

Uber built one of the most sophisticated data platforms in the industry. Their stack evolved:

```
Phase 1: MySQL + file-based ETL (outgrew quickly)
Phase 2: Hadoop + Vertica
Phase 3: Hive + HDFS + Spark
Phase 4: Custom real-time + Presto on HDFS
```

**Key innovations:**
- **Schemaless** — A sharded, fault-tolerant database built on top of MySQL
- **AresDB** — A GPU-powered real-time analytics database Uber built in-house (Go + GPU)
  - Uses GPU memory and parallel processing for sub-second analytical queries
  - Processes hundreds of millions of rows per second
- **Pinot** (contributed from LinkedIn) — Real-time OLAP for driver and trip analytics
- **Michelangelo** — Uber's internal ML platform for managing model training, serving, monitoring

**Why Uber built AresDB:** They needed real-time analytics on trip data with sub-second query latency. Existing tools couldn't combine the append-only time-series store with real-time aggregation at their scale.

**Lesson:** At extreme scale, you build your own tools. Uber's investment in AresDB, Michelangelo, and Schemaless shows that sometimes the right tool doesn't exist yet.

**Uber Engineering blog:** [eng.uber.com](https://eng.uber.com/uber-big-data-platform/)

---

## 3. LinkedIn — Kafka Origin & Samza

**Scale:** 1 billion+ members, hundreds of billions of messages/day across their platform

**Business problem:** LinkedIn's monolith was generating activity events (profile views, connection requests, job applications) that needed to flow to 40+ downstream consumers. Each consumer was pulling from the monolith — causing cascading failures.

### Why LinkedIn Invented Kafka

Before Kafka, LinkedIn used a custom message bus. Each producer had a direct pipe to each consumer. Adding a new consumer meant modifying the producer. The system was fragile.

**Kafka's core insight:** A distributed **commit log** where:
- Producers write to topics (one place)
- Any number of consumers can read from any offset
- Messages are retained for days/weeks (not deleted after consumption)
- Horizontal scaling via partitions

```
LinkedIn Activities (profile views, job applications, connections)
    ↓ Kafka Producers
Apache Kafka (distributed commit log)
    ↓ Consumer Groups (each reads independently)
├── Search Indexing (Elasticsearch)
├── Analytics (Hadoop/HDFS)
├── Notifications (Email/Push)
├── Feed Updates
├── LinkedIn Ads Platform
└── Apache Samza (real-time stream processing)
```

**Apache Samza** — LinkedIn's stream processing framework:
- Built specifically to integrate with Kafka
- Stateful processing with Kafka as the state backend
- Used for the LinkedIn feed, real-time metrics, and A/B testing

**ThirdEye** — LinkedIn's anomaly detection and root cause analysis tool:
- Monitors metrics across the platform
- Automatically detects anomalies in SLOs
- Routes alerts to on-call engineers

**Apache Calcite** — LinkedIn uses it in their metrics platform to implement a Lambda Architecture with a unified SQL layer on top of both batch (Hadoop) and real-time (Samza) data.

**Lesson:** Kafka was born from LinkedIn's need for a system that decoupled producers from consumers at massive scale. The log abstraction is the key insight — immutable, replayable, multi-consumer.

---

## 4. Spotify — Event Delivery to the Cloud

**Scale:** 400M+ users, 100M+ songs, billions of event records generated by music streaming behavior

**Business problem:** Move from on-premise event delivery infrastructure to Google Cloud Platform without dropping events, without taking the platform down, without rebuilding everything at once.

### The Journey (3-Part Series)

**Part I: The On-Premise System**

Spotify originally ran their event pipeline on bare metal servers in Stockholm. Events from clients (plays, skips, searches) were collected by:
1. A custom log collection agent on every server
2. Delivered to HDFS
3. Processed by Hadoop jobs
4. Results stored in Cassandra + PostgreSQL

**Problems:** Single datacenter = single point of failure. Scaling requires buying hardware. Operational burden massive.

**Part II: The Migration Plan**

Spotify migrated to Google Cloud Platform:
```
Client events → Kafka (on GCP) → Apache Beam on Dataflow
                                → Google Cloud Storage (GCS)
                                → BigQuery (analytics)
```

Key challenge: **Zero data loss** during migration. They ran old and new pipelines in parallel, validated counts matched, then cut over.

**Part III: Autoscaling**

Spotify built autoscaling for their Cloud Pub/Sub consumers — subscriber services that process events scale up/down based on message backlog, not CPU. This was a new concept for them (most autoscaling is CPU-based).

**Heroic** — Spotify's time-series database for internal metrics, built on top of Cassandra + Elasticsearch.

**Lesson:** Event delivery is critical infrastructure. Plan migrations carefully. Run old and new in parallel. Validate before cutover. Autoscaling consumers based on queue depth rather than CPU is the right model for streaming.

---

## 5. Airbnb — Druid & Spark Streaming

**Scale:** 150M+ guests, 4M+ hosts, operations in 220+ countries

**Business problem:** Airbnb's data team needed a way for business users to explore data interactively — filter by city, date, room type — with sub-second response times. Hive queries took minutes.

### Why Airbnb Chose Apache Druid

**Apache Druid** is a column-oriented distributed datastore optimized for:
- Sub-second OLAP queries on event data
- High-cardinality dimension filtering
- Time-series aggregation with rollups
- Real-time ingestion + historical data combined

```
Booking events → Kafka → Druid Real-Time Node (in-memory)
                        → Druid Historical Nodes (on deep storage S3)
                        ↓
                    Druid Broker (routes queries)
                        ↓
                    Business Intelligence Tools
```

**Spark Streaming for logging events:**
- Airbnb uses Spark Streaming to process user interaction events
- Events flow from their web services → Kafka → Spark Streaming → HDFS + Druid
- Allows them to see booking funnel analytics in near-real-time

**Data Infrastructure stack:**
- **Hive** — batch analytics on HDFS
- **Druid** — interactive OLAP dashboards
- **Superset (built at Airbnb!)** — open-source BI tool Airbnb created and donated to Apache

Airbnb built Apache Superset internally and open-sourced it — it became one of the most popular open-source BI tools.

**Lesson:** Different latency requirements need different tools. Hive for complex batch queries (hours). Druid for interactive sub-second exploration. Choose the right tool for the right SLA.

---

## 6. Pinterest — MySQL to S3 via Kafka

**Scale:** 450M+ monthly active users, 240 billion+ Pins

**Business problem:** Pinterest's MySQL databases were generating hundreds of terabytes of data. They needed to continuously replicate this data to S3/Hadoop for analytics without impacting production MySQL.

### Streaming MySQL to S3 via Kafka

```
MySQL (production)
    ↓ Kafka Connect (CDC from MySQL binlog)
Apache Kafka
    ↓ Kafka Streams (Kafka Consumer)
Amazon S3 (data lake in Parquet)
    ↓
Hadoop / Spark (analytics)
```

**Secor** — Pinterest's open-source Kafka-to-S3 consumer (later donated):
- Reads from Kafka topics
- Batches messages
- Writes to S3 in Parquet format with date partitioning
- Handles exactly-once delivery to S3

**Goku** — Pinterest's custom time-series database:
- Built when existing tools (InfluxDB, OpenTSDB) didn't scale to their needs
- Stores billions of data points for site metrics
- Sub-second query latency at Pinterest's scale

**Real-time Ads Platform with Kafka Streams:**
```
User action (pin, impression, click)
    ↓ Kafka
Kafka Streams (stateful processing)
    ↓
Real-time ad budget tracking (prevent overspend)
Real-time user action counting
```

**mysql_utils** — Pinterest's MySQL management tools (open-sourced on GitHub).

**Lesson:** CDC (Change Data Capture) from MySQL binlog is the right way to stream operational data to a data lake. Pinterest built Secor because they needed exactly-once S3 delivery — an important property for analytics correctness.

---

## 7. Twitter — Heron Stream Processing

**Scale:** 300M+ MAU, 500M+ tweets/day, real-time trending topics, ad targeting

**Business problem:** Twitter's original stream processing system (Apache Storm) was hitting scaling limits. Debugging failures was hard. Backpressure caused cascading failures. Operators couldn't isolate which topology was causing problems.

### Why Twitter Replaced Storm with Heron

**Apache Storm limitations at Twitter:**
- No isolation between topologies (one bad topology affects others)
- No backpressure mechanism
- Hard to profile performance bottlenecks
- Workers process multiple topologies on same JVM

**Apache Heron** — Twitter's Storm-compatible replacement:
- Each topology runs in isolated containers (no cross-topology impact)
- Built-in backpressure (slow downstream = slow upstream)
- Physical plan separate from logical plan
- Runs on Aurora (Twitter's cluster scheduler)

```
Tweet stream → Kafka → Heron Topology (topology = DAG of spouts + bolts)
                      ├── Spout (reads from Kafka)
                      ├── Bolt 1 (filter, transform)
                      ├── Bolt 2 (aggregate, join)
                      └── Bolt 3 (write to Cassandra/HDFS)
```

**Twitter's data infrastructure:**
- **Kafka** — central event bus (adopted after initial skepticism — blog post worth reading)
- **HBase** — distributed storage for tweet metadata
- **Manhattan** — Twitter's custom distributed key-value store (never open-sourced)
- **Heron** — stream processing (donated to Apache)

**Stats:** 500M+ tweets/day = ~6,000 tweets/second average, with massive spikes during events (Super Bowl, World Cup)

**Lesson:** Storm was fine for millions of events/day. At billions, the isolation boundary matters. Running multiple topologies in the same JVM is a false economy — one bad actor kills the whole machine.

---

## 8. Zalando — Delta Lake at Scale

**Scale:** 50M+ customers, Europe's largest online fashion platform, 100+ data engineering teams

**Business problem:** Zalando needed to support 100+ data engineering teams all writing to the same data lake on AWS S3. Concurrent writes without coordination caused data corruption. Schema changes broke downstream pipelines.

### Delta Lake as the Solution

**Without Delta Lake — problems:**
```
Team A writes → S3 bucket ← Team B reads (sees partial write!)
Schema change → All downstream consumers break immediately
Failed job → Corrupt data in S3 (no rollback mechanism)
```

**With Delta Lake:**
```
Team A writes (ACID transaction) → S3 + Delta Log
Team B reads → Always sees consistent snapshot
Schema evolution → Add columns safely with schema enforcement
Failed job → Automatic rollback to last good state
Time travel → SELECT * FROM table VERSION AS OF 5 (go back in time)
```

**Zalando's architecture:**
```
100+ teams each manage their own Delta tables
→ AWS S3 (data lake)
→ Delta Lake tables (ACID, schema evolution, time travel)
→ Databricks (Spark + Delta)
→ Structured Streaming (continuous pipelines)
→ Business Intelligence
```

**AWS CDK** — Zalando uses AWS Cloud Development Kit for infrastructure as code, defining their data lake infrastructure in Python.

**AWS Step Functions** — Used for orchestrating complex data workflows that span multiple services.

**Apache Flink** — Used for complex event generation and monitoring of business processes.

**Lesson:** Delta Lake solved the "multiple writers on S3" problem with ACID transactions. For organizations with many teams all writing to a shared data lake, this is transformative. The alternative is complicated distributed locking — Delta's log-based approach is elegant.

---

## 9. CERN — Apache Spark for Physics

**Scale:** Large Hadron Collider generates ~1 petabyte of data per second during collisions (hardware filtering reduces this to ~25 GB/s that's actually stored)

**Business problem:** Analyzing particle collision data to find the Higgs boson, dark matter candidates, and other fundamental physics. The data is incomprehensibly large and the signal-to-noise ratio is tiny.

### CERN's Data Engineering

**Data volume:** 
- LHC generates 600 million collisions/second
- Hardware triggers filter to ~100,000 interesting events/second
- Still results in ~25 GB/s of raw data to store and process
- Annual data storage: ~30 petabytes

**CERN accelerator logging service (built on Apache Spark):**
```
LHC accelerator devices (temperature, magnetic field, beam position)
    ↓ Thousands of sensors
Centralized logging service
    ↓ Apache Spark (batch + streaming)
HDFS / Cassandra storage
    ↓ Analysis
Physics results
```

**Apache Gobblin** — Used by CERN for data ingestion from heterogeneous sources into Hadoop.

**HDF5 (Hierarchical Data Format)** — Standard format for scientific data storage (multi-dimensional arrays, metadata).

**Anomaly detection with Spark:**
- CERN uses Spark Streaming to detect anomalies in their database infrastructure in real-time
- If a magnet's temperature spikes unexpectedly, alerts fire within seconds (preventing $1B+ equipment damage)

**Open Data:** CERN releases some of their collision data publicly at [opendata.cern.ch](http://opendata.cern.ch)

**Lesson:** The biggest data engineering challenges aren't in tech companies — they're in science. CERN's petabyte-scale processing requirements pushed the boundaries of what Spark and distributed systems can do.

---

## 10. Facebook/Meta — Spark at 60TB

**Scale:** 3B+ daily active users, 100+ petabytes of data processed daily

**Business problem:** Facebook had a 60TB production dataset that needed to be processed daily for feed ranking. The MapReduce jobs were taking too long and the compute cost was prohibitive.

### Migrating from MapReduce to Spark

**MapReduce problems for Facebook's use case:**
- Disk I/O for every map→reduce shuffle
- Can't pipeline multiple stages without writing to disk between each
- Python UDFs are slow (serialization overhead)
- Hard to express iterative ML algorithms

**Spark advantages:**
- In-memory operations for multi-stage pipelines
- RDD lineage for fault tolerance without disk checkpointing
- Native Python, SQL, and DataFrame APIs
- MLlib for the ML pipeline

**Result:** Facebook moved a 60TB daily batch job from MapReduce to Spark and got:
- **5-10x faster** execution
- **Significant cost reduction** (fewer machines, shorter runtime = less EC2 cost)

**Presto at Facebook:**
- Facebook built and open-sourced Presto — now used by thousands of companies
- At Facebook: used for interactive ad-hoc queries on HDFS data at petabyte scale
- Query latency: seconds (not Hive's minutes)

**Facebook's data infrastructure:**
- **Hive** — SQL-on-Hadoop (Facebook invented it and donated to Apache)
- **Presto** — interactive queries (Facebook built and open-sourced)
- **Scuba** — Facebook's internal time-series database (not open-sourced)
- **Giraph** — Graph processing (Facebook open-sourced for social graph analysis)

**Lesson:** The shift from MapReduce to Spark is one of the most important transitions in big data. Facebook's 60TB case study was an early public demonstration that Spark delivered on its performance promises at real production scale.

---

## 11. Booking.com — Kafka Ecosystem Management

**Scale:** 28M+ listings, operations in 230+ countries and territories

**Business problem:** Booking.com has 100+ microservices all producing events. Each service was using a different messaging system. This made it impossible to track events end-to-end and created operational chaos.

### Standardizing on Kafka

**Booking.com's approach:** Standardize the entire company on Apache Kafka as the unified event bus.

```
All Microservices → Apache Kafka (unified event bus)
                  → Confluent Platform (Schema Registry, connectors)
                  ↓
Data Engineering Team
├── Apache Flink (stream processing)
├── Apache Spark (batch processing)
└── Apache Druid (real-time OLAP)
```

**Challenges solved:**
- **Schema evolution** — Confluent Schema Registry enforces Avro schemas, prevents breaking changes
- **Multi-datacenter replication** — Kafka MirrorMaker replicates topics across datacenters
- **Consumer lag monitoring** — Burrow (open-sourced by LinkedIn) monitors consumer group lag

**Machine learning with Spark Streaming:**
Booking.com's ML team uses Spark Streaming to:
- Apply ML models to live booking intent signals
- Update feature stores in real-time
- A/B test ranking models with live traffic

**Lesson:** Standardizing on one messaging system (Kafka) dramatically reduces operational complexity when you have 100+ services. The Confluent Platform (Schema Registry, connectors) fills the gaps that vanilla Kafka leaves.

---

## 12. Lyft — Running Airflow at Scale

**Scale:** 40M+ riders, 3M+ drivers, operating in 600+ cities

**Business problem:** Lyft's data pipelines were a mix of cron jobs, custom scripts, and ad-hoc Python. Failure recovery required manual intervention. There was no visibility into pipeline state.

### Lyft's Airflow Setup

**Why Airflow:**
- Directed Acyclic Graph (DAG) model maps naturally to data pipeline dependencies
- Python code = version-controlled, code-reviewed pipelines
- Rich operator ecosystem (S3, Redshift, Spark, BigQuery, etc.)
- Scheduler retries, SLA monitoring, alerting built-in

**Lyft's production Airflow configuration:**
```python
# Lyft runs Airflow on Kubernetes
# CeleryExecutor with Redis as the broker
# 1000+ DAGs in production
# PostgreSQL as the metadata database (Amazon RDS)
```

**Key learnings from Lyft's Airflow blog post:**
1. **Dynamic DAGs** — Lyft generates DAGs programmatically for similar pipeline patterns
2. **SLA misses** — Set SLAs on critical pipelines and get paged when they miss
3. **DAG serialization** — Store DAGs in DB to avoid scheduler re-parsing overhead
4. **Pools** — Use Airflow pools to limit concurrent tasks on shared resources (Redshift connections)

**Amundsen (built at Lyft):**
- Data discovery and metadata catalog
- Shows table ownership, last updated, column descriptions, sample data
- Integrates with Hive, Presto, Redshift, Snowflake
- Donated to Linux Foundation

**Lesson:** Airflow works at Lyft's scale (1000+ DAGs) but requires careful configuration. Use CeleryExecutor for distribution, Redis for the broker, and RDS PostgreSQL for metadata. Build dynamic DAGs to reduce boilerplate.

---

## 13. ING Bank — Streaming Fraud Detection

**Scale:** 40M+ retail customers across Europe

**Business problem:** Credit card fraud detection. A fraudulent transaction must be flagged **before** it completes. Batch processing (detect fraud after the fact) is useless — you need millisecond latency decisions.

### Real-Time ML with Apache Flink

**The pipeline:**
```
Payment transaction event
    ↓ (< 100ms to complete)
Kafka (event buffer)
    ↓
Apache Flink (stateful stream processing)
├── Feature extraction (time since last transaction, amount, location)
├── Model scoring (pre-trained fraud model loaded into Flink)
├── Decision: approve / flag / block
    ↓
Payment decision returned to merchant
```

**The challenge: Updating ML models without downtime**

ING's innovation: **Adding models at runtime to a running Flink topology**

Traditional approach: Stop the Flink job, deploy new model, restart. Downtime = missed transactions.

ING's approach:
- Load models from a Kafka topic
- Flink operator subscribes to model updates
- New model replaces old model in-place
- Zero downtime model updates

```java
// Conceptual Flink operator with dynamic model loading
public class FraudModelOperator extends RichFlatMapFunction<Transaction, Decision> {
    private transient MLModel model;
    
    @Override
    public void open(Configuration config) {
        // Subscribe to model update Kafka topic
        modelUpdateStream.addSink(newModel -> this.model = newModel);
        this.model = loadInitialModel();
    }
    
    @Override
    public void flatMap(Transaction tx, Collector<Decision> out) {
        double fraudScore = model.predict(extractFeatures(tx));
        out.collect(new Decision(tx.id, fraudScore > 0.9 ? BLOCK : APPROVE));
    }
}
```

**Lesson:** When ML models need to be updated continuously (new fraud patterns emerge daily), you need a streaming-native approach to model serving. Flink's stateful operators make this possible without downtime.

---

## 14. Instagram — 300TB Spark Pipelines

**Scale:** 2B+ daily active users, 100M+ photos/videos uploaded daily

**Business problem:** Instagram's ML team needed to process 300TB of user interaction data daily to train recommendation models. MapReduce couldn't keep up. The pipeline was too slow and too expensive.

### Managing 300TB Spark Pipelines in Production

**Key challenges:**
1. **Cost** — Processing 300TB daily on Spark is expensive. Every minute of wasted compute costs money.
2. **Reliability** — With hundreds of Spark jobs, failures happen. Recovery must be automatic.
3. **Debugging** — When a job fails 4 hours in, you need to know why immediately.

**Instagram's solutions:**

**Checkpointing strategy:**
```python
# Checkpoint expensive computations so restarts resume mid-pipeline
df = spark.read.parquet("s3://bucket/raw/")
df_filtered = df.filter(df.created_at > yesterday)

# Checkpoint after expensive operation
df_filtered.write.mode("overwrite").parquet("s3://bucket/checkpoint/")
df_checkpoint = spark.read.parquet("s3://bucket/checkpoint/")

# Continue from checkpoint if earlier stages fail
df_features = df_checkpoint.transform(extract_features)
```

**Spot instance strategy:**
- Run Spark on AWS EC2 Spot instances (70-90% cheaper)
- Enable speculative execution to avoid stragglers on preempted nodes
- Checkpoint frequently so spot interruptions only lose minutes, not hours

**Lesson:** At 300TB/day, engineering discipline around checkpointing, spot instances, and monitoring is what separates a $5,000/day pipeline from a $50,000/day one. Every design decision has a cost attached.

---

## 15. Dropbox — Finding Kafka's Throughput Limit

**Business problem:** Dropbox uses Kafka for internal event streaming. They wanted to understand the maximum throughput their Kafka cluster could sustain before provisioning new capacity.

### Kafka Performance Testing

**What Dropbox measured:**
- Producer throughput vs. number of partitions
- Consumer throughput vs. consumer group size
- End-to-end latency at different load levels
- Impact of replication factor on write throughput

**Key findings:**
- Kafka throughput scales linearly with partitions (up to the network limit)
- Replication factor = 3 costs ~3x network bandwidth (worth it for durability)
- Producer batching dramatically improves throughput (batch.size = 64KB+ sweet spot)
- Consumer groups scale horizontally — 1 consumer per partition is the max parallelism

**Producer tuning for high throughput:**
```python
producer = KafkaProducer(
    bootstrap_servers=['kafka:9092'],
    acks='all',                    # Wait for all replicas (durability)
    compression_type='lz4',       # Compress batches (bandwidth savings)
    batch_size=65536,             # 64KB batch size (throughput)
    linger_ms=10,                 # Wait 10ms to fill batch (throughput)
    buffer_memory=33554432,       # 32MB producer buffer
    max_in_flight_requests_per_connection=5
)
```

**Lesson:** Understanding the performance characteristics of your messaging system is critical before you hit production limits. Test at expected scale + 10x headroom.

---

## 16. eBay — Spark as Core ETL Platform

**Scale:** 185M+ active buyers, 1.5B+ product listings

**Business problem:** eBay's data warehouse was built on traditional analytical DBMS systems. Moving to Apache Spark required migrating years of SQL logic and ensuring results matched exactly.

### Auto-Migration Framework

eBay built an **auto-migration framework** to translate SQL from their analytical DBMS dialect to Spark SQL:

```
Legacy DBMS SQL query
    ↓ Auto-migration framework
Spark SQL equivalent
    ↓ Validation layer (compares results)
Certified Spark query
```

**Key challenges:**
- **SQL dialect differences** — Window functions, date functions, and type coercions differ
- **Result validation** — Must verify outputs match exactly (checksum comparison)
- **Performance** — Spark must be at least as fast as the original DBMS

**Core ETL patterns that moved to Spark:**
```python
# Before (DBMS SQL): slow, expensive proprietary compute
# After (PySpark): fast, scalable, cost-effective

from pyspark.sql import functions as F

# Complex ETL: joining 3 large tables with window functions
result = (
    orders
    .join(customers, "customer_id")
    .join(products, "product_id")
    .withColumn("running_total",
        F.sum("revenue").over(
            Window.partitionBy("customer_id")
                  .orderBy("order_date")
        )
    )
    .filter(F.col("order_date") >= "2024-01-01")
    .groupBy("customer_id", "category")
    .agg(F.sum("revenue").alias("total_revenue"),
         F.countDistinct("order_id").alias("order_count"))
)
```

**Lesson:** Migration from legacy DBMS to Spark is possible at scale, but requires automated translation and rigorous validation. The business can't accept "approximately correct" — results must match exactly.

---

## 17. Disney — Auto-Scaling Kinesis Streams

**Business problem:** Disney Streaming (Disney+, ESPN+, Hulu) needs to handle massive traffic spikes — Super Bowl, Star Wars premiere, Mandalorian episode drops. Traffic can spike 10-100x in seconds.

### Auto-Scaling Kinesis with Lambda

**The architecture:**
```
Disney+ Client (video play event) → AWS Kinesis Streams
                                  → AWS Lambda (auto-triggered per shard)
                                  ↓
                               S3 (raw events)
                               Redshift (analytics)
```

**The challenge:** Kinesis shards have fixed throughput (1MB/s write per shard). During a spike, you need more shards. During normal hours, fewer shards = lower cost.

**Auto-scaling solution:**
```python
import boto3

def scale_kinesis_stream(stream_name, target_shards):
    kinesis = boto3.client('kinesis', region_name='us-east-1')
    
    current = kinesis.describe_stream_summary(StreamName=stream_name)
    current_shards = current['StreamDescriptionSummary']['OpenShardCount']
    
    if target_shards > current_shards:
        kinesis.split_shard(
            StreamName=stream_name,
            ShardToSplit=get_most_loaded_shard(stream_name),
            NewStartingHashKey=get_midpoint_hash(stream_name)
        )
    elif target_shards < current_shards:
        kinesis.merge_shards(
            StreamName=stream_name,
            ShardToMerge=get_adjacent_shards(stream_name)
        )
```

**CloudWatch alarm triggers the Lambda** — when bytes/second exceeds threshold → scale up. When it drops → scale down.

**Lesson:** Serverless event streaming (Kinesis + Lambda) can auto-scale to handle entertainment industry spikes without pre-provisioning massive infrastructure that sits idle 99% of the time.

---

## 18. Expedia — Spark Streaming + Kafka Best Practices

**Scale:** 200+ travel booking sites, complex multi-step booking funnels

**Business problem:** Track users through the booking funnel in real-time. Detect abandonment. Trigger retargeting. Understand conversion rates within minutes, not hours.

### Kafka + Spark Streaming Best Practices

**Expedia's hard-won lessons:**

1. **Use Direct API (not Receiver-based)** for exactly-once semantics:
```python
# Receiver-based (at-least-once — avoid for financial data)
# Direct API (exactly-once — use this)
from pyspark.streaming.kafka import KafkaUtils

direct_stream = KafkaUtils.createDirectStream(
    ssc,
    topics=["booking-events"],
    kafkaParams={"bootstrap.servers": "kafka:9092",
                 "group.id": "booking-analytics"}
)
```

2. **Tune batch duration** for your latency/throughput tradeoff:
   - 1 second batches = low latency, high overhead
   - 30 second batches = high latency, efficient processing
   - 5 second batches = Expedia's sweet spot for their use case

3. **Checkpointing** — checkpoint to HDFS/S3, not local disk:
```python
ssc.checkpoint("s3://expedia-streaming/checkpoints/booking-analytics/")
```

4. **Monitor** consumer lag continuously — if lag grows, scale out consumers.

**Lesson:** Kafka + Spark Streaming is not plug-and-play. You need to tune batch duration, choose the right API (Direct vs Receiver), checkpoint correctly, and monitor lag. The defaults are not production-ready.

---

## 19. NASA — Apache Science Data Analytics Platform

**Scale:** Petabytes of satellite imagery, rover data, telescope observations

**Business problem:** NASA's scientists needed to analyze multi-petabyte Earth observation datasets. Traditional tools couldn't scale. Each science team built their own ad-hoc solution.

### Apache SDAP (Science Data Analytics Platform)

**NASA built SDAP to:**
- Provide a unified platform for analyzing Earth science data
- Enable ocean scientists to query sea surface temperature across decades
- Allow atmospheric scientists to correlate satellite readings with ground sensors

**Technologies:**
- **Apache Spark** — distributed computation across satellite imagery datasets
- **HDF5** — standard scientific data format for storing multidimensional arrays
- **Apache Cassandra** — metadata storage for dataset catalog
- **Solr** — search interface for scientific datasets

**Mars Science Laboratory (Curiosity Rover):**
- Curiosity transmits ~100MB/day from Mars via the Deep Space Network
- Data arrives at JPL, gets processed with Python scripts + Spark
- Scientists analyze it to plan next day's activities

**OnSight** — AR visualization system for Curiosity team:
- Scientists "walk" on Mars surface using AR headsets
- Data powering this visualization is processed by Spark pipelines

**Lesson:** Science is one of the original big data domains. NASA's scale challenges are unique (petabyte satellite archives) but the tools (Spark, Cassandra, HDF5) are the same ones commercial DE uses. "Big data" was always a science problem first.

---

## 20. Woot/Amazon — Serverless Data Lake on AWS

**Business problem:** Woot.com (Amazon subsidiary, deal-of-the-day site) needed a data lake without a dedicated data engineering team to manage infrastructure.

### Serverless Data Lake Architecture

**"We wanted a data lake without a data lake team"**

```
Woot transaction data → AWS DMS (Database Migration Service)
                      → Amazon S3 (raw data lake)
                      → AWS Glue (ETL jobs, managed Spark)
                      → AWS Glue Data Catalog (Hive-compatible metadata)
                      → Amazon Athena (serverless SQL on S3)
                      → Amazon QuickSight (BI dashboards)
```

**Why serverless:**
- No servers to provision or manage
- Pay only for queries run (Athena charges per TB scanned)
- Glue automatically scales ETL jobs
- DMS handles CDC replication automatically

**Cost at Woot's scale:** This architecture runs for hundreds of dollars/month, not thousands.

**Athena query optimization:**
```sql
-- Partition pruning: only scan relevant partitions
SELECT product_id, SUM(revenue) AS total_revenue
FROM woot_transactions
WHERE year = 2024 AND month = 12  -- Partition filter
AND category = 'Electronics'
GROUP BY product_id
ORDER BY total_revenue DESC
LIMIT 100;
-- Without partition filter: scans 100TB → costs $500
-- With partition filter: scans 50GB → costs $0.25
```

**Lesson:** If you don't have a dedicated data engineering team, go serverless. AWS Glue + Athena + S3 is a legitimate production data lake that requires near-zero operational overhead.

---

## 21. Slack — Streaming Data Pipelines

**Scale:** 20M+ daily active users, billions of messages, complex real-time features

**Business problem:** Slack's notification system, search index, and analytics all need events as they happen. Batch processing would make notifications delayed and search stale.

### Streaming Pipeline Architecture

```
Slack message sent → Internal event bus (Kafka)
                   → Notification service (push, email)
                   → Search indexing (Elasticsearch)
                   → Analytics pipeline (Hadoop → data warehouse)
                   → ML features (real-time features for smart replies)
```

**Key challenges:**
- **Ordering** — Messages in a channel must appear in order to users
- **At-least-once delivery** — A missed notification is worse than a duplicate
- **Scale spikes** — Monday mornings spike 5-10x vs Friday evenings

**Lesson:** Notification systems are one of the most demanding streaming use cases — failures are immediately visible to users. Invest in exactly-once delivery and comprehensive monitoring.

---

## 22. Drivetribe — Kappa Architecture with Flink

**Business problem:** Drivetribe (automotive social network) needed to process user interactions (views, likes, posts) both historically and in real-time for content recommendations.

### Why They Chose Kappa Over Lambda

**Lambda Architecture problems:**
- Two code paths: batch (Spark) + streaming (Flink) for the same logic
- Both must produce identical results → maintenance burden
- When the batch path changes, you must also change the streaming path

**Kappa Architecture (Drivetribe's choice):**
```
All events → Kafka (retained for 60+ days)
           → Apache Flink (one code path handles both batch replay + real-time)
           → Feature store / recommendations
```

**Batch reprocessing in Kappa:**
When you need to reprocess historical data:
1. Start a new Flink job from Kafka offset 0
2. Process all historical events through the same Flink logic
3. Output goes to a new destination
4. Cut over when new job catches up to real-time

**Lesson:** Kappa Architecture is simpler than Lambda when your stream processing engine (Flink) is powerful enough to handle both batch-like reprocessing and real-time processing. The single code path is a significant maintenance advantage.

---

## 23. Tinder — Scalable Monitoring with Spark

**Business problem:** Tinder generates billions of swipe events daily. Monitoring this at scale required aggregating millions of metric series with sub-minute latency.

### Scalable Monitoring Architecture

```
Tinder app events → Kafka → Apache Spark (aggregation)
                          → TimescaleDB / InfluxDB (metrics storage)
                          → Grafana (visualization)
                          → PagerDuty (alerting)
```

**Tinder's Spark monitoring pipeline:**
- Reads raw metrics from Kafka
- Applies sliding window aggregations (5-minute, 1-hour, 1-day)
- Writes aggregated metrics to time-series database
- Grafana dashboards query the TSDB for live display

**Why Spark for monitoring (not Flink or Kafka Streams):**
- Tinder already had Spark expertise
- Batch micro-windows (30-second batches) was acceptable latency
- Spark's rich ecosystem of connectors to output stores

**Lesson:** You don't always need the "best" tool — you need the tool your team knows. Tinder's Spark-for-monitoring choice was pragmatic. They had Spark expertise, and Spark was good enough. "Good enough" ships; "perfect" delays.

---

## 24. Grammarly — Versatile Analytics on Spark

**Business problem:** Grammarly analyzes text from 20M+ daily active users for grammatical errors, style suggestions, and plagiarism. This requires both batch ML model training and real-time serving.

### Building a Versatile Analytics Pipeline

**Grammarly's unified pipeline:**
```
User documents (text)
    ↓ Real-time processing
Apache Kafka
    ↓
Apache Spark (unified batch + streaming)
├── Feature extraction (NLP features from text)
├── ML model training (weekly batch)
├── A/B test analysis
└── Analytics aggregation
    ↓
Databricks (managed Spark)
```

**Key insight:** By using Spark for both ML training and analytics, Grammarly's data scientists and data engineers share the same infrastructure, reducing hand-off friction.

**Lesson:** When both ML and analytics teams use the same compute platform (Spark), data sharing, model validation, and feature engineering become dramatically simpler.

---

## 25. PayPal — Scalable Event Pipeline

**Scale:** 400M+ active accounts, 20M+ merchants, $1.4T+ payment volume/year

**Business problem:** PayPal processes millions of payment events per second. Each payment triggers a cascade of events: fraud check, compliance logging, notification, accounting, analytics.

### Event-Driven Architecture at PayPal

```
Payment transaction
    ↓ Apache Kafka (multi-datacenter)
    ├── Fraud Detection (real-time ML scoring)
    ├── Compliance (immutable audit log)
    ├── Notifications (email/push)
    ├── Analytics (Hadoop → data warehouse)
    └── Accounting (ledger updates)
```

**PayPal's financial data requirements:**
- **Exactly-once** — A payment event must update the ledger exactly once (not twice, not zero)
- **Durability** — Kafka replication factor = 3, across 3 availability zones
- **Ordering** — All events for a given account must be processed in order

**Heroku + Salesforce event pipeline:**
- PayPal also builds event pipelines connecting Salesforce to Heroku
- Event-driven microservices communicate via Kafka topics

**Lesson:** Financial data engineering has the strictest requirements of any domain. Exactly-once delivery, cross-datacenter replication, and strict ordering are non-negotiable. These requirements justify significant engineering investment.

---

## 26. Key Lessons Across All Case Studies

After studying 25 companies, these patterns emerge repeatedly:

### Lesson 1: Kafka is Everywhere

Every company at scale eventually adopts Kafka (or a managed equivalent). The reasons:
- Decoupling producers from consumers
- Replayability (events stored for days/weeks)
- Horizontal scaling via partitions
- Multi-consumer fan-out

### Lesson 2: The MapReduce → Spark Transition is Universal

Every company that started with Hadoop MapReduce migrated to Spark. The reasons:
- In-memory processing (10-100x faster for iterative jobs)
- Unified batch, streaming, SQL, and ML in one engine
- Python API (PySpark) — data scientists can contribute

### Lesson 3: One Tool Per Use Case

Companies routinely run multiple tools for the same data:
- **Hive** for ad-hoc exploration (minutes, cheapest)
- **Presto/Trino** for interactive queries (seconds, moderate cost)
- **Druid/Pinot** for dashboards (sub-second, cached)
- **Flink** for real-time decisions (milliseconds, always-on)

Choosing one tool for all use cases is a compromise. The right architecture has the right tool for each SLA.

### Lesson 4: Build Your Own Only When You Must

Netflix built custom ML tools. Uber built AresDB. Pinterest built Goku. Twitter built Heron. These were all cases where existing tools couldn't scale.

**Before building custom:** exhaust open-source options. Custom tools are expensive to build, maintain, and staff.

### Lesson 5: Batch First, Streaming Later

Every company started with batch (daily Hadoop jobs). Real-time streaming came later, when the business need justified the operational complexity.

**Advice:** Start with batch. Add streaming when you have a concrete latency requirement you can't meet with batch.

### Lesson 6: Lambda → Kappa is a Natural Evolution

Companies start with Lambda (batch + streaming in parallel). As streaming matures, they move to Kappa (one streaming path handles both). Drivetribe is the clearest example, but this pattern repeats.

---

## 27. Architecture Patterns Summary

| Company | Problem | Solution | Key Tool |
|---------|---------|----------|----------|
| Netflix | 1.3PB/day event analytics | Batch + real-time dual pipeline | Kafka + Spark Streaming + Cassandra |
| Uber | Real-time pricing analytics | GPU-accelerated OLAP | AresDB (custom) |
| LinkedIn | Event bus for 40+ consumers | Distributed commit log | Apache Kafka (invented here) |
| Spotify | On-prem → cloud migration | Staged migration with validation | Beam/Dataflow + Pub/Sub |
| Airbnb | Sub-second OLAP for business | Time-series OLAP store | Apache Druid |
| Pinterest | MySQL replication to S3 | CDC via Kafka | Kafka + Secor (custom) |
| Twitter | Storm couldn't scale | Container-isolated stream processing | Apache Heron |
| Zalando | 100 teams writing to same lake | ACID data lake | Delta Lake |
| CERN | Petabyte physics data | Distributed scientific analytics | Spark + HDF5 |
| Facebook | 60TB batch too slow | MapReduce → Spark migration | Apache Spark |
| Booking.com | 100+ microservice events | Unified event bus | Kafka + Confluent |
| Lyft | Ad-hoc cron jobs | Declarative pipeline scheduling | Apache Airflow |
| ING | Fraud in milliseconds | Real-time ML with live model updates | Apache Flink |
| Instagram | 300TB/day ML pipeline | Cost-optimized Spark on Spot | PySpark + EC2 Spot |
| Dropbox | Kafka capacity planning | Systematic load testing | Apache Kafka |
| eBay | DBMS → Spark migration | Auto-SQL translation framework | Apache Spark |
| Disney | Traffic spikes 100x | Auto-scaling event streams | Kinesis + Lambda |
| Expedia | Booking funnel tracking | Tuned streaming pipeline | Kafka + Spark Streaming |
| NASA | Petabyte science archives | Unified science analytics | Spark + HDF5 + Cassandra |
| Woot | No DE team, need data lake | Fully serverless analytics | Glue + Athena + S3 |
| Slack | Real-time notifications | Event-driven pipeline | Kafka + Elasticsearch |
| Drivetribe | Lambda maintenance burden | Single streaming code path | Apache Flink (Kappa) |
| Tinder | Billions of swipe metrics | Spark-based monitoring | Spark + InfluxDB + Grafana |
| Grammarly | ML + analytics silos | Unified Spark platform | Databricks + Spark |
| PayPal | Exactly-once payment events | Multi-AZ Kafka + strict ordering | Apache Kafka |

---

← [10 Tools Catalog](10-tools-catalog.md) | [12 Best Practices →](12-best-practices.md)
