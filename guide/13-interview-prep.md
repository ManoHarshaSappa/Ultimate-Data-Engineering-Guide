# Data Engineering Interview Prep: 100+ Q&As

**Author:** Mano Harsha Sappa | [LinkedIn](https://www.linkedin.com/in/manoharshasappa/) | [Portfolio](https://manoharshasappa.github.io/portfolio_ManoHarshaSappa/) | [Email](mailto:sappamanoharsha@gmail.com)

---

## Contents

1. [How to Prepare for DE Interviews](#1-how-to-prepare-for-de-interviews)
2. [Python & Programming](#2-python--programming)
3. [SQL](#3-sql)
4. [APIs & Integration](#4-apis--integration)
5. [Apache Kafka & Message Queues](#5-apache-kafka--message-queues)
6. [Apache Spark & Distributed Processing](#6-apache-spark--distributed-processing)
7. [Data Warehousing & Modeling](#7-data-warehousing--modeling)
8. [Data Lakes & Lakehouse](#8-data-lakes--lakehouse)
9. [Orchestration & Workflow (Airflow)](#9-orchestration--workflow-airflow)
10. [Docker & Kubernetes](#10-docker--kubernetes)
11. [Cloud Platforms (AWS, GCP, Azure)](#11-cloud-platforms-aws-gcp-azure)
12. [System Design Questions](#12-system-design-questions)
13. [Behavioral & Situational Questions](#13-behavioral--situational-questions)
14. [Interview Checklist](#14-interview-checklist)

---

## 1. How to Prepare for DE Interviews

### What Companies Actually Test

| Round | What They Test | How to Prepare |
|-------|----------------|----------------|
| Phone screen | Basic DE concepts, experience | Know your resume deeply, practice explaining projects |
| SQL round | Window functions, CTEs, optimization | LeetCode SQL (Medium), practice on real datasets |
| Coding round | Python ETL, data manipulation, Pandas | Build small ETL scripts daily |
| System design | Architecture, scalability, trade-offs | Study case studies, draw diagrams |
| Technical deep-dive | Your resume projects in detail | Be able to explain every technical choice you made |

### The STAR Answer Framework

Use STAR for behavioral questions:
- **S**ituation: Context (brief)
- **T**ask: What you were responsible for
- **A**ction: What YOU specifically did (use "I", not "we")
- **R**esult: Quantified outcome (10x faster, saved $50K, reduced latency from 2h to 5min)

### The 3 Types of DE Interview Questions

1. **Conceptual** — "What is the difference between Kafka and RabbitMQ?"
2. **Practical** — "Write a PySpark job to find the top 10 customers by revenue"
3. **Design** — "Design a real-time fraud detection pipeline for a bank"

---

## 2. Python & Programming

**Q1: What is Apache Spark and how is it used with Python?**
> Apache Spark is a distributed data processing framework with in-memory computing. PySpark is its Python API, letting engineers write Spark applications in Python. PySpark supports SQL, DataFrames, ML, and streaming. It's typically used when data is too large for a single machine — think 10GB+ files, or processing terabytes in parallel.

**Q2: How do you handle schema changes in a data pipeline?**
> Several strategies:
> 1. **Schema evolution** with Avro + Schema Registry — add/remove optional fields without breaking consumers
> 2. **Backward compatibility** — new consumers can read old data (add fields with defaults)
> 3. **Forward compatibility** — old consumers can read new data (ignore unknown fields)
> 4. **Schema registry** enforces compatibility mode (BACKWARD, FORWARD, FULL)
> 5. In practice: version your schemas (`orders.v1`, `orders.v2`), sunset old schemas after migration

**Q3: How do you ensure data quality in pipelines?**
> Layered approach:
> - **At ingestion:** schema validation, reject malformed records to dead-letter queue
> - **At transformation:** dbt tests (not_null, unique, accepted_values, relationships)
> - **Post-load:** row count comparison with source, anomaly detection on business metrics
> - **Ongoing:** Great Expectations or Soda for continuous monitoring, freshness checks, alerting on anomalies

**Q4: Explain data partitioning and why it matters.**
> Partitioning divides data into smaller, independently-addressable chunks. Benefits:
> - **Query performance** — partition pruning skips irrelevant partitions entirely (reading 1/365th of a yearly table)
> - **Parallel processing** — each partition processed by a different worker
> - **Lifecycle management** — delete entire partitions (drop yesterday's raw data)
> 
> Best practice: Partition by the most common filter column — almost always `date` or `event_date`.

**Q5: What are common challenges with distributed processing?**
> 1. **Data skew** — some keys have far more data, causing one executor to process 10x more than others. Fix: salting, broadcast joins, AQE (Spark 3+)
> 2. **Fault tolerance** — nodes die mid-job. Fix: checkpointing, idempotent writes
> 3. **Network shuffles** — expensive data movement for groupBy/joins. Fix: minimize shuffles, repartition strategically
> 4. **Consistency** — partial writes. Fix: Delta Lake ACID transactions, idempotent pipelines
> 5. **Latency vs throughput tradeoff** — tuning batch size, parallelism, micro-batch duration

**Q6: Write a Python function to paginate through an API.**
```python
import requests
import time

def fetch_all_records(base_url: str, headers: dict, page_size: int = 100):
    """Fetch all records from a paginated REST API."""
    all_records = []
    page = 1
    
    while True:
        response = requests.get(
            base_url,
            headers=headers,
            params={"page": page, "per_page": page_size},
            timeout=30
        )
        response.raise_for_status()
        data = response.json()
        
        records = data.get("results", [])
        all_records.extend(records)
        
        # Check if we've reached the last page
        if len(records) < page_size or not data.get("next"):
            break
        
        page += 1
        time.sleep(0.1)  # Rate limiting courtesy
    
    return all_records
```

**Q7: How do you handle PII in a data pipeline?**
> 1. **Classify data** — identify which fields are PII (email, SSN, IP address, etc.)
> 2. **Minimize collection** — don't collect PII you don't need (GDPR data minimization)
> 3. **Mask/hash** at source — hash emails with SHA-256 before landing in data lake
> 4. **Encrypt** — SSE-KMS for data at rest, TLS for data in transit
> 5. **Access control** — only authorized roles can query raw PII columns
> 6. **Right to erasure** — design deletion into the pipeline architecture from day one
> 7. **Audit logging** — track every access to PII-containing tables

**Q8: What is the difference between `map()`, `filter()`, and `reduce()` in Python?**
```python
numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

# map: apply function to every element → same length output
doubled = list(map(lambda x: x * 2, numbers))
# [2, 4, 6, 8, 10, 12, 14, 16, 18, 20]

# filter: keep only elements where function returns True → smaller output
evens = list(filter(lambda x: x % 2 == 0, numbers))
# [2, 4, 6, 8, 10]

# reduce: combine all elements into one value → single output
from functools import reduce
total = reduce(lambda acc, x: acc + x, numbers)
# 55
```

---

## 3. SQL

**Q9: Write a query to find the top 3 customers by revenue in each country.**
```sql
WITH customer_revenue AS (
    SELECT
        c.country,
        c.customer_id,
        c.customer_name,
        SUM(o.revenue) AS total_revenue,
        RANK() OVER (
            PARTITION BY c.country
            ORDER BY SUM(o.revenue) DESC
        ) AS revenue_rank
    FROM orders o
    JOIN customers c ON o.customer_id = c.customer_id
    GROUP BY c.country, c.customer_id, c.customer_name
)
SELECT country, customer_name, total_revenue, revenue_rank
FROM customer_revenue
WHERE revenue_rank <= 3
ORDER BY country, revenue_rank;
```

**Q10: Explain window functions and give examples.**
> Window functions perform calculations across a set of rows related to the current row, without collapsing them into a single output row.
```sql
SELECT
    order_id,
    customer_id,
    order_date,
    revenue,
    
    -- Ranking
    ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date) AS order_number,
    RANK() OVER (ORDER BY revenue DESC) AS revenue_rank,
    DENSE_RANK() OVER (ORDER BY revenue DESC) AS dense_rank,
    
    -- Aggregation over a window
    SUM(revenue) OVER (PARTITION BY customer_id ORDER BY order_date 
                       ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total,
    AVG(revenue) OVER (PARTITION BY customer_id 
                       ORDER BY order_date
                       ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS rolling_7day_avg,
    
    -- Lead/Lag
    LAG(revenue, 1) OVER (PARTITION BY customer_id ORDER BY order_date) AS prev_order_revenue,
    LEAD(order_date, 1) OVER (PARTITION BY customer_id ORDER BY order_date) AS next_order_date,
    
    -- Percentile
    PERCENT_RANK() OVER (ORDER BY revenue) AS percentile
    
FROM orders
ORDER BY customer_id, order_date;
```

**Q11: Write a query to implement SCD Type 2 in SQL.**
```sql
-- Detect changed records and close old versions
UPDATE dim_customers AS target
SET
    valid_to = NOW() - INTERVAL '1 second',
    is_current = FALSE
FROM staging_customers AS source
WHERE target.customer_id = source.customer_id
  AND target.is_current = TRUE
  AND (
      target.email     <> source.email OR
      target.address   <> source.address OR
      target.phone     <> source.phone
  );

-- Insert new versions for changed records
INSERT INTO dim_customers (customer_id, email, address, phone, valid_from, valid_to, is_current)
SELECT
    source.customer_id,
    source.email,
    source.address,
    source.phone,
    NOW() AS valid_from,
    '9999-12-31'::DATE AS valid_to,
    TRUE AS is_current
FROM staging_customers source
WHERE EXISTS (
    SELECT 1 FROM dim_customers target
    WHERE target.customer_id = source.customer_id
      AND target.is_current = FALSE
      AND target.valid_to = NOW() - INTERVAL '1 second'
);

-- Insert brand new customers
INSERT INTO dim_customers (customer_id, email, address, phone, valid_from, valid_to, is_current)
SELECT
    source.customer_id,
    source.email,
    source.address,
    source.phone,
    NOW(),
    '9999-12-31'::DATE,
    TRUE
FROM staging_customers source
WHERE NOT EXISTS (
    SELECT 1 FROM dim_customers target
    WHERE target.customer_id = source.customer_id
);
```

**Q12: What is the difference between RANK, DENSE_RANK, and ROW_NUMBER?**
```sql
-- Given revenues: 100, 100, 80, 60

SELECT revenue,
    ROW_NUMBER()  OVER (ORDER BY revenue DESC) AS row_num,   -- 1, 2, 3, 4
    RANK()        OVER (ORDER BY revenue DESC) AS rank_val,  -- 1, 1, 3, 4 (gap after tie)
    DENSE_RANK()  OVER (ORDER BY revenue DESC) AS dense_rank -- 1, 1, 2, 3 (no gap)
FROM orders;
```

**Q13: How do you optimize a slow SQL query?**
> Step-by-step approach:
> 1. **EXPLAIN / EXPLAIN ANALYZE** — see the query plan, identify table scans vs index scans
> 2. **Add indexes** — on JOIN columns and WHERE filter columns
> 3. **Partition pruning** — ensure WHERE clause includes the partition column
> 4. **Reduce data early** — filter in subqueries/CTEs before joining
> 5. **Avoid `SELECT *`** — read only columns you need
> 6. **Avoid functions on indexed columns** — `DATE(created_at)` prevents index use; use range instead
> 7. **Materialized views** — pre-compute expensive aggregations
> 8. **Query cache** — for identical repeated queries

**Q14: Write a query to find users who purchased in consecutive months.**
```sql
WITH monthly_purchases AS (
    SELECT
        customer_id,
        DATE_TRUNC('month', order_date) AS purchase_month
    FROM orders
    GROUP BY customer_id, DATE_TRUNC('month', order_date)
),
consecutive AS (
    SELECT
        customer_id,
        purchase_month,
        LAG(purchase_month) OVER (
            PARTITION BY customer_id ORDER BY purchase_month
        ) AS prev_month
    FROM monthly_purchases
)
SELECT DISTINCT customer_id
FROM consecutive
WHERE purchase_month = prev_month + INTERVAL '1 month'
ORDER BY customer_id;
```

**Q15: What is the difference between INNER JOIN, LEFT JOIN, and FULL OUTER JOIN?**
```
INNER JOIN:      Only rows with matching keys in BOTH tables
LEFT JOIN:       All rows from LEFT, NULLs for non-matching RIGHT
RIGHT JOIN:      All rows from RIGHT, NULLs for non-matching LEFT
FULL OUTER JOIN: All rows from BOTH, NULLs where no match on either side
CROSS JOIN:      Cartesian product — every row × every row (dangerous at scale)
```

---

## 4. APIs & Integration

**Q16: What is the difference between REST and SOAP?**
> **REST** (Representational State Transfer):
> - Uses HTTP methods (GET, POST, PUT, DELETE)
> - Stateless — each request contains all info needed
> - Returns JSON or XML, typically JSON
> - Lightweight, fast, industry standard for modern APIs
> 
> **SOAP** (Simple Object Access Protocol):
> - Protocol with strict XML envelope format
> - Stateful (can maintain session state)
> - Built-in WS-Security, WS-ReliableMessaging
> - Used in enterprise/banking where formal contracts are required
> 
> For data engineering: 99% of APIs you'll call are REST/JSON.

**Q17: What is OAuth 2.0 and how does it work in data pipelines?**
> OAuth 2.0 is an authorization framework that lets an application access resources on behalf of a user without seeing their password.
```
Flow for DE pipelines (Client Credentials Grant):

1. Your pipeline → Authorization Server (client_id + client_secret)
2. Auth Server → Access token (JWT, expires in 1 hour)
3. Your pipeline → API (Bearer: {access_token})
4. API → Data (if token valid and has required scope)
```
```python
import requests

def get_oauth_token(auth_url, client_id, client_secret, scope):
    response = requests.post(auth_url, data={
        "grant_type": "client_credentials",
        "client_id": client_id,
        "client_secret": client_secret,
        "scope": scope
    })
    return response.json()["access_token"]

token = get_oauth_token("https://auth.api.com/token", CLIENT_ID, CLIENT_SECRET, "read:orders")
response = requests.get("https://api.com/orders", headers={"Authorization": f"Bearer {token}"})
```

**Q18: What is a webhook and how does it differ from polling?**
> **Polling:** Your system asks "any new data?" on a schedule (every 5 min). Simple but wasteful.
> **Webhook:** The source system pushes data to you when an event happens. Efficient but requires a public endpoint.
> 
> **When to use each:**
> - Polling: Legacy systems with no webhook support, batch daily ingestion
> - Webhooks: Real-time requirements, event-driven architectures
```python
# Webhook receiver (Flask)
from flask import Flask, request, jsonify
import hmac, hashlib

app = Flask(__name__)

@app.route('/webhook/orders', methods=['POST'])
def receive_order_webhook():
    # Verify the webhook signature (prevent spoofing)
    signature = request.headers.get('X-Signature-256')
    body = request.get_data()
    expected = hmac.new(WEBHOOK_SECRET.encode(), body, hashlib.sha256).hexdigest()
    
    if not hmac.compare_digest(f"sha256={expected}", signature):
        return jsonify({"error": "Invalid signature"}), 401
    
    order = request.get_json()
    process_order_async(order)  # Don't block — process asynchronously
    return jsonify({"status": "accepted"}), 202  # Return fast
```

**Q19: How do you handle API rate limits?**
```python
import time
import requests
from functools import wraps

def with_retry_and_backoff(max_retries=5, backoff_factor=2):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(max_retries):
                response = func(*args, **kwargs)
                
                if response.status_code == 200:
                    return response
                elif response.status_code == 429:  # Rate limited
                    retry_after = int(response.headers.get("Retry-After", 60))
                    print(f"Rate limited. Waiting {retry_after}s...")
                    time.sleep(retry_after)
                elif response.status_code >= 500:   # Server error — retry
                    wait = backoff_factor ** attempt
                    print(f"Server error {response.status_code}. Retrying in {wait}s...")
                    time.sleep(wait)
                else:
                    response.raise_for_status()  # 4xx client errors — don't retry
            
            raise Exception(f"Failed after {max_retries} attempts")
        return wrapper
    return decorator
```

---

## 5. Apache Kafka & Message Queues

**Q20: What is Apache Kafka and what problem does it solve?**
> Kafka is a distributed event streaming platform (distributed commit log). It solves the N-to-N integration problem: without Kafka, M producers connecting to N consumers requires M×N connections. With Kafka, each producer writes to topics and each consumer reads independently — M+N connections. Additionally, Kafka retains messages (default 7 days), enabling replay, multiple consumers at different offsets, and fault recovery.

**Q21: Explain the Kafka architecture.**
```
Cluster: Multiple brokers
Broker: One Kafka server
Topic: Named stream of records (like a database table)
Partition: Ordered, immutable log segment within a topic
Offset: Position of a message within a partition
Producer: Writes messages to topics
Consumer: Reads messages from topics
Consumer Group: Group of consumers sharing a topic's partitions
  → Each partition is consumed by exactly ONE consumer per group
ZooKeeper/KRaft: Cluster coordination (ZK being replaced by KRaft in newer versions)
```

**Q22: What is a consumer group and why does it matter?**
> A consumer group is a set of consumers that share the work of consuming a topic. Each partition is assigned to exactly one consumer in the group. This enables horizontal scaling: add more consumers = more throughput. Maximum parallelism = number of partitions. If consumers > partitions, some consumers are idle.
```
Topic: orders (6 partitions)
Consumer Group A (analytics service): 3 consumers
  → Consumer 1: partitions 0, 1
  → Consumer 2: partitions 2, 3
  → Consumer 3: partitions 4, 5

Consumer Group B (email service): 6 consumers
  → Each consumer: 1 partition (max parallelism)

Both groups read ALL messages independently — neither affects the other.
```

**Q23: What is the difference between at-least-once, at-most-once, and exactly-once?**
| Guarantee | How | Risk | Use Case |
|-----------|-----|------|----------|
| **At-most-once** | Auto-commit before processing | Data loss (missed messages) | Non-critical analytics |
| **At-least-once** | Commit after processing | Duplicates (process twice on failure) | Most DE pipelines (use idempotent writes) |
| **Exactly-once** | Transactional producer + idempotent consumer | Complex, performance cost | Financial transactions, billing |

**Q24: How do you handle message ordering in Kafka?**
> Ordering is guaranteed **within a partition**, not across partitions.
> - Use the same **key** for all messages that must be ordered (e.g., `customer_id`)
> - All messages with the same key → same partition → guaranteed order
> - For global ordering: use 1 partition (but loses parallelism — use only for low-volume streams)
```python
# Ensure customer orders are processed in order per customer
producer.send(
    "orders",
    key=order["customer_id"].encode(),  # Same customer → same partition
    value=order
)
```

**Q25: What is the Kafka Schema Registry and why use it?**
> Schema Registry stores and enforces Avro/JSON/Protobuf schemas for Kafka messages. It:
> - Prevents breaking schema changes from reaching consumers
> - Enforces compatibility modes (BACKWARD, FORWARD, FULL)
> - Stores schemas by subject (one per topic or topic-value/topic-key)
> - Serializes schemas efficiently (send schema ID, not full schema, in each message)

**Q26: How do you monitor Kafka consumer health?**
> The key metric is **consumer lag**: how many messages are in the topic that the consumer hasn't processed yet.
> - Lag = 0: Consumer is caught up
> - Lag growing: Consumer can't keep up — scale out or fix processing bottleneck
> - Lag > threshold: Trigger alert, page on-call
> 
> Tools: Kafka's built-in consumer group commands, Burrow (LinkedIn), Kafdrop, Confluent Control Center.

---

## 6. Apache Spark & Distributed Processing

**Q27: What is the difference between Spark and MapReduce?**
| Aspect | MapReduce | Apache Spark |
|--------|-----------|--------------|
| Computation model | Disk-based (reads/writes between each stage) | In-memory (chain stages without disk) |
| Speed | Baseline | 10-100x faster for iterative jobs |
| API | Java only | Python, Scala, Java, R |
| Programming model | Map → Shuffle → Reduce | DAG of transformations |
| Streaming | No native support | Spark Streaming / Structured Streaming |
| ML | No native support | MLlib |
| SQL | Hive (on top of MR) | Spark SQL (native) |

**Q28: What is the difference between RDDs, DataFrames, and Datasets?**
> **RDD (Resilient Distributed Dataset):** Low-level API. Collection of objects distributed across cluster. Type-safe in Java/Scala, untyped in Python. No optimizer. Use only for low-level operations.
> 
> **DataFrame:** Distributed table with named columns and schema. High-level API. Catalyst optimizer generates efficient execution plan. Best for SQL-like data manipulation.
> 
> **Dataset:** DataFrame + compile-time type safety (Scala/Java only). Not available in PySpark.
> 
> **Modern guidance:** Use DataFrames in PySpark. RDDs only when DataFrames can't express the operation.

**Q29: Explain Spark's lazy evaluation and why it matters.**
> In Spark, transformations (`filter`, `select`, `groupBy`) build a logical plan but don't execute. Only **actions** (`count`, `show`, `write`, `collect`) trigger execution.
```python
# This builds a logical plan (no data moves)
df1 = spark.read.parquet("s3://bucket/orders/")
df2 = df1.filter(df1.revenue > 100)
df3 = df2.groupBy("customer_id").agg(F.sum("revenue"))

# This triggers execution (ALL transformations run together)
df3.write.parquet("s3://bucket/result/")
```
> **Why it matters:** Spark's Catalyst optimizer can see the ENTIRE chain and optimize it (e.g., push filters down closer to the source, choose better join strategies). This couldn't happen if each transformation executed immediately.

**Q30: What is data skew and how do you fix it?**
> Data skew occurs when data is unevenly distributed across partitions. Symptom: one Spark task takes 10x longer than all others. Root cause: some key values appear far more often than others (e.g., "null" customer_id, or one massive customer).
```python
# Fix 1: Salt the join key (add random prefix)
SALT = 20
df_large = df_large.withColumn(
    "salted_key", F.concat(
        (F.rand() * SALT).cast("int").cast("string"),
        F.lit("_"),
        F.col("customer_id")
    )
)
# Expand small table to match all salt values
df_small = df_small.withColumn(
    "salt_range", F.array([F.lit(i) for i in range(SALT)])
).withColumn("salt", F.explode("salt_range")) \
 .withColumn("salted_key", F.concat(F.col("salt").cast("string"), F.lit("_"), F.col("customer_id")))

result = df_large.join(df_small, "salted_key")

# Fix 2: Use Spark 3 AQE (Adaptive Query Execution)
# Just enable it — Spark handles skew automatically
spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")
```

**Q31: Write a PySpark job to compute the 7-day rolling average revenue per customer.**
```python
from pyspark.sql import SparkSession
from pyspark.sql import functions as F
from pyspark.sql.window import Window

spark = SparkSession.builder.appName("rolling-average").getOrCreate()

orders = spark.read.parquet("s3://bucket/orders/")

window_spec = (
    Window
    .partitionBy("customer_id")
    .orderBy(F.col("order_date").cast("long"))
    .rangeBetween(
        -6 * 86400,  # 6 days ago in seconds
        Window.currentRow
    )
)

result = orders.withColumn(
    "rolling_7d_avg_revenue",
    F.avg("revenue").over(window_spec)
).select(
    "customer_id",
    "order_date",
    "revenue",
    "rolling_7d_avg_revenue"
)

result.write.mode("overwrite").partitionBy("order_date").parquet("s3://bucket/output/")
```

---

## 7. Data Warehousing & Modeling

**Q32: What is Kimball dimensional modeling?**
> Kimball's methodology organizes data into facts (measurements) and dimensions (context). Facts contain numeric measurements and foreign keys to dimensions. Dimensions contain descriptive attributes.
> 
> **Star schema:** Fact table surrounded by dimension tables. One join to get full context.
> **Snowflake schema:** Dimensions are normalized (dimensions of dimensions). More space-efficient, harder to query.
> 
> **Kimball says:** Prefer star schema (denormalized) for analytical workloads — joins are expensive in analytics.

**Q33: What is the difference between Fact tables and Dimension tables?**
| Aspect | Fact Table | Dimension Table |
|--------|-----------|-----------------|
| Contains | Measurements, metrics, events | Descriptive attributes, context |
| Key types | Foreign keys + degenerate keys | Surrogate key + natural key |
| Examples | `fct_orders`, `fct_pageviews` | `dim_customers`, `dim_products`, `dim_date` |
| Row count | Very high (billions) | Lower (millions max) |
| Update pattern | Append-only | Slowly changing |
| Examples of values | revenue, quantity, duration | name, address, category, status |

**Q34: What are Slowly Changing Dimensions (SCD)?**
> SCDs handle how dimension attributes change over time.
> 
> **SCD Type 1:** Overwrite. No history. `UPDATE SET address = new_address`. Use when history doesn't matter.
> 
> **SCD Type 2:** New row for each change. Full history preserved with `valid_from`, `valid_to`, `is_current`. Use for everything that matters analytically.
> 
> **SCD Type 3:** Add a `previous_value` column. Partial history. Rarely used.
```sql
-- SCD Type 2 dimension
SELECT * FROM dim_customers
WHERE customer_id = 'C123'
ORDER BY valid_from;

-- customer_id | email           | city      | valid_from  | valid_to    | is_current
-- C123        | old@email.com   | Austin    | 2022-01-01  | 2023-06-14  | FALSE
-- C123        | new@email.com   | Seattle   | 2023-06-15  | 9999-12-31  | TRUE
```

**Q35: What is the difference between a Data Warehouse and a Data Lake?**
| Aspect | Data Warehouse | Data Lake | Lakehouse |
|--------|---------------|-----------|-----------|
| Schema | Schema-on-write (defined before loading) | Schema-on-read (defined at query time) | Both |
| Data types | Structured only | All types (structured, semi, unstructured) | All types |
| Processing | ETL (transform before loading) | ELT (load raw, transform later) | ELT |
| Cost | Expensive (compute + storage bundled) | Cheap (object storage) | Cheap storage + elastic compute |
| Examples | Redshift, Snowflake, BigQuery | S3 + Hive, HDFS | Delta Lake, Iceberg, Hudi |
| Query performance | Fast (optimized columnar) | Slow on raw (optimized with tables) | Fast |

**Q36: What is a surrogate key and why use it instead of natural keys?**
> A surrogate key is a system-generated unique identifier (usually integer or UUID) with no business meaning. Natural keys are meaningful business identifiers (customer email, SSN, product SKU).
> 
> **Use surrogate keys because:**
> - Natural keys can change (email changes, SKU gets renamed)
> - Natural keys can be NULL (no SSN for international customers)
> - Natural keys can be long strings (expensive to join on)
> - Surrogate keys are stable — history preserved even if business key changes
```sql
-- Surrogate key in dbt using md5 hash
{{ dbt_utils.generate_surrogate_key(['customer_id', 'valid_from']) }}
```

---

## 8. Data Lakes & Lakehouse

**Q37: What is the Medallion Architecture?**
> A pattern that organizes data lake layers by data quality:
> - **Bronze** (raw): Exact copy of source data, never modified. Append-only. Full history.
> - **Silver** (cleaned): Validated, deduplicated, standardized. Business rules applied. Not yet business-specific.
> - **Gold** (business): Business-ready dimensional models, aggregates, and feature tables for specific use cases.
> 
> Why: Separates ingestion concerns from transformation concerns. Silver layer is reusable across multiple Gold use cases.

**Q38: What is Delta Lake and what problems does it solve?**
> Delta Lake is an open-source storage layer that adds ACID transactions to object storage (S3, ADLS, GCS). Problems it solves:
> 1. **Partial writes** — without ACID, a failed job leaves partial data. Delta uses a transaction log to make writes atomic.
> 2. **Concurrent writes** — multiple writers cause data corruption. Delta uses optimistic concurrency control.
> 3. **Schema enforcement** — accidentally writing wrong schema fails cleanly instead of corrupting data.
> 4. **Time travel** — read data as it was at a point in time (for debugging, auditing).
> 5. **Upserts/Merges** — MERGE INTO command for CDC-style incremental updates.

**Q39: What is the difference between Delta Lake, Apache Iceberg, and Apache Hudi?**
| Feature | Delta Lake | Apache Iceberg | Apache Hudi |
|---------|-----------|----------------|------------|
| Created by | Databricks | Netflix | Uber |
| Best with | Spark, Databricks | Any engine (Spark, Flink, Trino) | Spark (especially streaming) |
| Key strength | Databricks ecosystem, Z-ORDER | Engine agnosticism, hidden partitioning | Record-level upserts, streaming |
| Time travel | Yes | Yes | Yes |
| Schema evolution | Yes | Yes | Yes |
| ACID | Yes | Yes | Yes |

---

## 9. Orchestration & Workflow (Airflow)

**Q40: What is Apache Airflow and what problem does it solve?**
> Airflow is a workflow orchestration platform that lets you programmatically author, schedule, and monitor data pipelines as Directed Acyclic Graphs (DAGs). Problems it solves:
> - Replace fragile cron jobs with dependency-aware, retry-capable, monitored pipelines
> - Visualize pipeline state (what's running, what failed, what's waiting)
> - Alert on failures and SLA misses
> - Enable backfilling historical data with a single command
> - Manage thousands of pipelines in one place

**Q41: What is a DAG in Airflow?**
> A DAG (Directed Acyclic Graph) is a collection of tasks with their dependencies. "Directed" means dependencies flow in one direction. "Acyclic" means no circular dependencies (A→B→A is invalid).
```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime, timedelta

with DAG(
    dag_id="orders_pipeline",
    start_date=datetime(2024, 1, 1),
    schedule_interval="0 6 * * *",   # Daily at 6 AM
    catchup=False,
    default_args={"retries": 2, "retry_delay": timedelta(minutes=5)}
) as dag:
    
    extract = PythonOperator(task_id="extract_orders", python_callable=extract_fn)
    transform = PythonOperator(task_id="transform_orders", python_callable=transform_fn)
    load = PythonOperator(task_id="load_to_warehouse", python_callable=load_fn)
    
    # Define dependencies: extract → transform → load
    extract >> transform >> load
```

**Q42: What is the difference between Airflow, Prefect, and Dagster?**
| Aspect | Airflow | Prefect | Dagster |
|--------|---------|---------|---------|
| Paradigm | Task-centric DAGs | Flow-centric (Python functions) | Asset-centric (data assets) |
| Dynamic tasks | Hard (needs dynamic DAG generation) | Easy (`.map()` over any iterable) | Yes |
| Local development | Hard (needs full stack) | Easy (local runner) | Easy (local runner) |
| Data lineage | Not built-in | Not built-in | Built-in (asset graph) |
| Best for | Large, established teams | Python-first teams | Data assets + lineage |

**Q43: What is Airflow's XCom?**
> XCom (Cross-Communication) allows Airflow tasks to exchange small amounts of data. Tasks push values to XCom and downstream tasks pull them.
```python
def extract(**context):
    records = fetch_from_api()
    # Push to XCom
    context['task_instance'].xcom_push(key='record_count', value=len(records))
    return records  # Implicit XCom push of return value

def validate(**context):
    # Pull from XCom
    count = context['task_instance'].xcom_pull(
        task_ids='extract_task',
        key='record_count'
    )
    assert count > 0, "No records found!"
```
> **Warning:** XCom is stored in Airflow's metadata DB. Don't push large DataFrames — use shared storage (S3, GCS) instead and pass the path via XCom.

---

## 10. Docker & Kubernetes

**Q44: What is Docker and why do data engineers use it?**
> Docker packages an application + all its dependencies into a container. For data engineers:
> - **Reproducibility** — "works on my machine" → "works in any container"
> - **Dependency isolation** — Python 3.9 + pandas 2.0 + spark 3.5 in one container, without conflicting with other projects
> - **Deployment** — containers run identically in dev, staging, and prod
> - **Local development** — run Kafka + Postgres + Airflow locally with one command

**Q45: What is the difference between a Docker image and a container?**
> **Image:** Blueprint. Read-only template with the application and its dependencies. Like a class.
> **Container:** Running instance of an image. Has its own filesystem, process space, and network. Like an object (instance of a class).

**Q46: What is Kubernetes and when would a data engineer use it?**
> Kubernetes (K8s) is a container orchestration system that manages deployment, scaling, and lifecycle of containers across a cluster. Data engineers use it to:
> - Run Airflow at scale (MWAA or self-managed with CeleryExecutor on K8s)
> - Run Spark jobs on K8s (Spark on Kubernetes replaces YARN)
> - Deploy and scale streaming consumers (Kafka consumers, Flink jobs)
> - Run containerized ML training and serving

---

## 11. Cloud Platforms (AWS, GCP, Azure)

**Q47: What are the core AWS services for data engineering?**
| Service | Category | Description |
|---------|----------|-------------|
| **S3** | Storage | Object storage (data lake foundation) |
| **Redshift** | Warehouse | Columnar MPP data warehouse |
| **Glue** | ETL | Managed Spark-based ETL service |
| **EMR** | Compute | Managed Hadoop/Spark cluster |
| **Kinesis** | Streaming | Managed event streaming (Kafka alternative) |
| **Lambda** | Serverless | Event-triggered compute functions |
| **Athena** | Query | Serverless SQL on S3 |
| **DMS** | Migration | Database migration + CDC |
| **MSK** | Kafka | Managed Kafka on AWS |
| **MWAA** | Airflow | Managed Apache Airflow |
| **Step Functions** | Orchestration | State machine workflow orchestration |

**Q48: What is BigQuery and what makes it different?**
> BigQuery is Google's fully-managed, serverless data warehouse. Key differentiators:
> - **Separation of storage and compute** — scale each independently, pay per query
> - **Columnar storage** (Capacitor format) — optimized for analytical queries
> - **Automatic partitioning and clustering** — optimize without manual indexes
> - **BQML** — train ML models with SQL
> - **Streaming inserts** — ingest events in real-time directly to BigQuery
> - **Cost model** — $5/TB scanned (vs Redshift's hourly cluster cost) — partition your tables!

**Q49: What is the difference between Kinesis and Kafka?**
| Aspect | AWS Kinesis | Apache Kafka |
|--------|------------|--------------|
| Management | Fully managed | Self-managed (or Confluent/MSK) |
| Scaling | Scale shards (manual or auto) | Add brokers (manual or auto) |
| Retention | Max 7 days (365 days with extended) | Configurable (days to forever) |
| Ordering | Per shard | Per partition |
| Consumer model | Poll + checkpoint in DynamoDB | Consumer group + offset |
| Cost | Pay per shard hour + PUT units | Infrastructure cost + operational effort |
| Best for | AWS-native, simple streaming | Polycloud, high throughput, long retention |

---

## 12. System Design Questions

**Q50: Design a real-time fraud detection pipeline.**
```
Requirements:
- Credit card transactions arriving at 10,000/second
- Flag fraudulent transactions before authorization completes (< 100ms)
- Store all transactions for 7 years (compliance)
- Retrain fraud model weekly

Architecture:

Payment Gateway → Kafka (topic: transactions, 20 partitions)
                ↓
         Apache Flink (stateful stream processor)
         ├── Feature extraction (time since last tx, amount, location, velocity)
         ├── Model scoring (pre-loaded fraud model via broadcast state)
         ├── Rule engine (hardcoded rules: > $10K in 10 min = suspicious)
         └── Decision: APPROVE / FLAG / BLOCK (< 50ms per transaction)
                ↓
         Decision → Payment Gateway (complete authorization)
         Transaction + Features → S3 (compliance archive)
         Flagged transactions → Alerts queue → Fraud analyst review
         All transactions → Delta Lake (Gold layer, for model retraining)
                ↓
         Weekly batch: Spark on EMR reads Delta Lake
         → Retrain model (XGBoost, LightGBM)
         → Validate on holdout (compare AUC to current model)
         → Deploy new model → Flink broadcast state update (zero downtime)

Key decisions:
- Flink not Spark Streaming: need true millisecond latency, not micro-batches
- Stateful Flink: velocity features require state (transactions per account per hour)
- Kafka 20 partitions: 20 parallel Flink task managers = 10K tx/sec throughput
- Delta Lake for training: time travel enables reproducible model training datasets
```

**Q51: Design a daily data pipeline for an e-commerce company.**
```
Requirements:
- Ingest orders, users, products from 5 source systems
- Transform into dimensional model
- Available by 8 AM UTC for business reporting

Architecture:

Source systems (Postgres, MySQL, Salesforce, Stripe, Shopify)
    ↓
[6 AM UTC] Airbyte OR Fivetran (managed ELT, 300+ connectors)
    ↓
S3 Bronze layer (raw, partitioned by ingestion date)
    ↓
[6:30 AM UTC] dbt (transform via Airflow DAG trigger)
    ├── Staging models (stg_postgres__orders, stg_stripe__charges)
    ├── Intermediate (int_orders__with_payments)
    ├── Dimension models (dim_customers, dim_products, dim_date)
    └── Fact models (fct_orders, fct_payments)
    ↓
Snowflake / BigQuery (Gold layer)
    ↓
[8 AM UTC] BI Tools (Tableau, Looker, Metabase)
    ↓ (email to executives)

Monitoring:
- Airflow SLA = 7:30 AM (alert if DAG not done by 7:30)
- Elementary: freshness check (all gold tables < 4 hours old)
- Row count anomaly detection: > 20% deviation from 7-day average
```

**Q52: How would you design a data lake for a media streaming company?**
```
Scale: 500M users, 100B events/day, 500TB new data/day

Architecture:

Event Collection:
Client SDKs → Kafka (50 partitions) → Schema Registry (Avro)

Streaming Layer:
Kafka → Spark Structured Streaming
     → Bronze S3 (raw Avro, partitioned by event_date/event_hour)

Batch Processing (nightly):
Bronze → dbt on Spark (Silver: deduplicated, validated)
Silver → dbt (Gold: fct_streams, fct_searches, dim_content, dim_users)

File format: Parquet + Snappy, columnar, ~1GB per file (avoid small files)
Table format: Apache Iceberg (engine-agnostic, hidden partitioning)
Catalog: AWS Glue Data Catalog (Hive-compatible)

Query Engines:
- Trino (interactive ad-hoc) → data analysts
- Spark (batch jobs) → data engineers / ML
- BigQuery (business reporting) → business analysts

Data Quality:
- Great Expectations: check play event counts, duration ranges
- Monte Carlo / Elementary: anomaly detection on business metrics
```

---

## 13. Behavioral & Situational Questions

**Q53: Tell me about a time you improved a slow data pipeline.**
> *(Example STAR answer)*
> **Situation:** Our daily revenue report was taking 4 hours to complete, causing it to miss the 6 AM business SLA consistently.
> **Task:** I was responsible for diagnosing and fixing the pipeline without breaking the downstream reports.
> **Action:** I profiled the pipeline using Spark UI and found a single groupBy aggregation causing a 200-way shuffle on an unevenly distributed `customer_id` column. 15% of customers had 60% of the data. I added Spark's Adaptive Query Execution (AQE) and salted the join keys for the top 100 customers.
> **Result:** Pipeline time dropped from 4 hours to 38 minutes. The report now consistently finishes by 3 AM, giving 3 hours of buffer before the SLA.

**Q54: Describe a time data was wrong in production. How did you handle it?**
> *(Framework for your answer)*
> - **Detect** — how was it caught? (monitoring? user report? accident?)
> - **Contain** — what did you do immediately? (stop the pipeline? notify users?)
> - **Fix** — what was the root cause? how did you fix the data?
> - **Prevent** — what monitoring or tests did you add to prevent recurrence?

**Q55: How do you handle disagreements with a stakeholder about data requirements?**
> Focus on: understanding their underlying need vs their stated requirement, bringing data to the discussion (show what's feasible with examples), proposing phased delivery, and getting written alignment before building. The goal isn't to "win" — it's to build the right thing.

**Q56: How do you stay up to date with the data engineering field?**
> Mention: specific newsletters (Data Eng Weekly, dbt Weekly Wrap), podcasts (Data Engineering Podcast), communities (Data Talks Club, DataExpert.io Discord), following specific engineers on LinkedIn, reading engineering blogs (Netflix Tech Blog, Uber Engineering, Databricks Blog).

---

## 14. Interview Checklist

### Before the Interview

- [ ] Review your resume projects — be able to explain every technical choice
- [ ] Practice SQL window functions on a real dataset (NYC Taxi or similar)
- [ ] Build a small PySpark job from scratch (no IDE autocomplete practice)
- [ ] Draw a data architecture from memory for a use case you know
- [ ] Prepare 3 STAR stories: one success, one failure, one conflict

### Common Questions by Role Level

**Entry Level / Junior:**
- Write SQL window functions
- Explain ETL vs ELT
- What is Kafka? What is partitioning?
- Write a Python script to read CSV, clean, and load to DB

**Mid-Level:**
- Design a daily batch pipeline end to end
- Debug a slow Spark job
- Implement SCD Type 2 in dbt
- Explain Delta Lake vs Iceberg

**Senior / Staff:**
- Design a real-time fraud detection system
- Design a multi-tenant data platform
- How would you migrate from on-prem Hadoop to cloud lakehouse?
- How do you architect for global data residency requirements?

### Questions to Ask the Interviewer

- "What does the data engineering team's on-call rotation look like?"
- "How do you currently handle schema changes between producer teams?"
- "What's the biggest data quality challenge the team is facing?"
- "What's the tech stack and what tools are you considering changing?"
- "How is data engineering work prioritized against product engineering?"

---

← [12 Best Practices](12-best-practices.md) | [14 Books & Resources →](14-books-resources.md)
