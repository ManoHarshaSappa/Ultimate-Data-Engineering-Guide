# 04 — Advanced Data Engineering Skills

> **Guide by [Mano Harsha Sappa](https://www.linkedin.com/in/manoharshasappa/)** | [Portfolio](https://manoharshasappa.github.io/portfolio_ManoHarshaSappa/) | [Back to Index](../README.md)

---

## Contents

- [Apache Kafka — Distributed Streaming](#apache-kafka--distributed-streaming)
- [Apache Spark — Distributed Processing](#apache-spark--distributed-processing)
- [MapReduce — The Foundation](#mapreduce--the-foundation)
- [HDFS — Hadoop Distributed File System](#hdfs--hadoop-distributed-file-system)
- [Apache Flink — True Stream Processing](#apache-flink--true-stream-processing)
- [REST APIs — The Connect Layer](#rest-apis--the-connect-layer)
- [Apache NiFi — Data Flow Automation](#apache-nifi--data-flow-automation)
- [Machine Learning in Production](#machine-learning-in-production)

---

## Apache Kafka — Distributed Streaming

Apache Kafka is the industry standard for high-throughput, fault-tolerant event streaming. Originally built at LinkedIn to handle billions of events per day, Kafka is now used by Netflix, Uber, Twitter, Spotify, and thousands of other companies.

### Why a Message Queue?

Without a message queue, your data pipeline looks like this:
```
Source → Direct call → Consumer
```

Problems: If the consumer is slow or down, data is lost. If the source is too fast, the consumer is overwhelmed.

With Kafka:
```
Source → Kafka Topic → Consumer (at its own pace)
```

Benefits:
- **Decoupling**: Source and consumer don't know about each other
- **Buffering**: Kafka absorbs traffic spikes
- **Replay**: Consumers can re-read past messages
- **Fan-out**: Multiple consumers can read the same topic independently

### Kafka Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    KAFKA CLUSTER                            │
│                                                             │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐              │
│  │  Broker 1  │  │  Broker 2  │  │  Broker 3  │  (Nodes)   │
│  │           │  │           │  │           │              │
│  │ Topic A   │  │ Topic A   │  │ Topic A   │              │
│  │ Part. 0   │  │ Part. 1   │  │ Part. 2   │              │
│  │ (Leader)  │  │ (Leader)  │  │ (Leader)  │              │
│  │           │  │           │  │           │              │
│  │ Topic B   │  │ Topic B   │  │           │              │
│  │ Part. 0   │  │ Part. 0   │  │           │              │
│  │ (Replica) │  │ (Leader)  │  │           │              │
│  └───────────┘  └───────────┘  └───────────┘              │
│                                                             │
│              ZooKeeper / KRaft (coordination)               │
└─────────────────────────────────────────────────────────────┘
         ↑ Producers write                 ↓ Consumers read
```

### Key Kafka Concepts

**Topic**: A named stream of records. Like a database table name.

**Partition**: A topic is split into partitions for parallel processing. Each partition is an ordered, immutable log of messages.

**Offset**: A message's position within a partition (0, 1, 2, 3...). Consumers track their offset.

**Consumer Group**: Multiple consumers sharing the workload of reading from a topic. Each partition is read by exactly one consumer in the group.

**Broker**: A Kafka server. A cluster has multiple brokers for fault tolerance.

**Replication Factor**: How many copies of each partition exist. RF=3 means 3 copies → can survive 2 broker failures.

**Retention**: How long messages are kept. Default is 7 days. You can set to "indefinite" for event sourcing.

### How to Produce and Consume Messages

#### Python Producer
```python
from kafka import KafkaProducer
import json

producer = KafkaProducer(
    bootstrap_servers=['localhost:9092'],
    value_serializer=lambda v: json.dumps(v).encode('utf-8'),
    key_serializer=lambda k: k.encode('utf-8') if k else None,
    acks='all',                    # Wait for all replicas to acknowledge
    retries=3,
    compression_type='gzip'
)

# Send message
producer.send(
    topic='user-events',
    key='user_123',                # Key determines which partition
    value={
        'user_id': 'user_123',
        'event': 'purchase',
        'amount': 49.99,
        'timestamp': '2024-01-15T10:30:00Z'
    }
)
producer.flush()  # Ensure all messages are sent
producer.close()
```

#### Python Consumer
```python
from kafka import KafkaConsumer
import json

consumer = KafkaConsumer(
    'user-events',
    bootstrap_servers=['localhost:9092'],
    group_id='analytics-consumer',         # Consumer group
    auto_offset_reset='earliest',          # Start from beginning if no offset
    enable_auto_commit=True,               # Auto-commit offsets
    value_deserializer=lambda v: json.loads(v.decode('utf-8'))
)

for message in consumer:
    print(f"Partition: {message.partition}, Offset: {message.offset}")
    event = message.value
    print(f"User: {event['user_id']}, Event: {event['event']}")
    # Process the event...
```

### Kafka CLI Commands

```bash
# Start Zookeeper
docker run -d --name zookeeper \
    --network data-network \
    -e ALLOW_ANONYMOUS_LOGIN=yes \
    bitnami/zookeeper:latest

# Start Kafka
docker run -d --name kafka \
    --network data-network \
    -e KAFKA_CFG_ZOOKEEPER_CONNECT=zookeeper:2181 \
    -e ALLOW_PLAINTEXT_LISTENER=yes \
    bitnami/kafka:latest

# Create topic
docker exec kafka kafka-topics.sh \
    --create \
    --bootstrap-server localhost:9092 \
    --topic user-events \
    --partitions 6 \
    --replication-factor 1

# List topics
docker exec kafka kafka-topics.sh \
    --list \
    --bootstrap-server localhost:9092

# Describe topic (see partition details)
docker exec kafka kafka-topics.sh \
    --describe \
    --topic user-events \
    --bootstrap-server localhost:9092

# Produce test messages
docker exec -it kafka kafka-console-producer.sh \
    --broker-list localhost:9092 \
    --topic user-events

# Consume messages
docker exec -it kafka kafka-console-consumer.sh \
    --bootstrap-server localhost:9092 \
    --topic user-events \
    --from-beginning

# Consumer group status
docker exec kafka kafka-consumer-groups.sh \
    --bootstrap-server localhost:9092 \
    --describe \
    --group analytics-consumer
```

### Kafka Best Practices

1. **Partitioning strategy**: Use a meaningful key (user_id, device_id) for ordering guarantees within a partition
2. **Replication factor**: Always RF≥3 in production
3. **Consumer groups**: Scale consumers by adding instances (up to partition count)
4. **Message retention**: Set longer than your consumers' SLA (if pipeline is down for 2 days, you need 2+ days of retention)
5. **Schema registry**: Use Confluent Schema Registry with Avro for schema evolution
6. **Monitoring**: Track consumer lag (offset behind producer) — high lag = slow consumer

---

## Apache Spark — Distributed Processing

Apache Spark is the dominant framework for processing large-scale data. It replaced MapReduce as the standard for batch processing and also supports streaming.

### Spark vs MapReduce

| Feature | MapReduce | Apache Spark |
|---------|-----------|-------------|
| Processing | Disk-based | In-memory |
| Speed | Slow (disk I/O between stages) | 10-100x faster |
| Flexibility | Map + Reduce only | Any computation |
| Languages | Java | Python, Scala, Java, R |
| Streaming | Not supported | Yes (Spark Streaming) |
| SQL | Hive on top | SparkSQL built-in |
| ML | External (Mahout) | MLlib built-in |

### How Spark Works

```
┌──────────────────────────────────────────────────────┐
│                    SPARK CLUSTER                      │
│                                                       │
│  ┌─────────────┐                                     │
│  │   DRIVER    │  ← Your Python/Scala code runs here │
│  │  SparkContext│                                     │
│  └──────┬──────┘                                     │
│         │ distributes work                            │
│  ┌──────▼──────────────────────────────────────────┐ │
│  │              CLUSTER MANAGER                     │ │
│  │         (YARN / Kubernetes / Standalone)         │ │
│  └──────┬──────────────────────────────────────────┘ │
│         │                                            │
│  ┌──────▼──┐  ┌─────────┐  ┌─────────┐             │
│  │Executor │  │Executor │  │Executor │  (Workers)   │
│  │ Tasks   │  │ Tasks   │  │ Tasks   │              │
│  └─────────┘  └─────────┘  └─────────┘             │
└──────────────────────────────────────────────────────┘
```

**Driver**: Coordinates the job, creates execution plan  
**Executors**: Do the actual computation in parallel  
**SparkContext**: Entry point, connects driver to cluster

### RDDs vs DataFrames vs Datasets

**RDDs (Resilient Distributed Datasets)** — Low-level API
```python
# Old way — still useful for custom operations
rdd = sc.textFile("hdfs://path/to/data.csv")
rdd_filtered = rdd.filter(lambda line: "ERROR" in line)
error_count = rdd_filtered.count()
```

**DataFrames** — Modern, high-level API (Recommended)
```python
from pyspark.sql import SparkSession
from pyspark.sql import functions as F

spark = SparkSession.builder \
    .appName("MyPipeline") \
    .config("spark.sql.shuffle.partitions", "200") \
    .getOrCreate()

# Read data
df = spark.read.parquet("s3://bucket/raw/events/")

# Transformations (lazy — nothing runs yet)
result = df \
    .filter(F.col("event_type") == "purchase") \
    .withColumn("revenue_usd", F.col("amount") * F.col("exchange_rate")) \
    .groupBy("user_id", F.date_trunc("day", F.col("timestamp")).alias("date")) \
    .agg(
        F.sum("revenue_usd").alias("daily_revenue"),
        F.count("*").alias("purchase_count")
    ) \
    .orderBy("date", "daily_revenue", ascending=[True, False])

# Action (triggers execution)
result.write.parquet("s3://bucket/processed/daily_revenue/", mode="overwrite")
```

### Key PySpark Operations

```python
from pyspark.sql import functions as F
from pyspark.sql.window import Window

# Schema inspection
df.printSchema()
df.describe().show()
df.count()

# Filtering
df.filter(F.col("amount") > 100)
df.where("status = 'active' AND age >= 18")

# Column operations
df.withColumn("full_name", F.concat(F.col("first_name"), F.lit(" "), F.col("last_name")))
df.withColumn("created_date", F.to_date(F.col("created_at")))
df.drop("unnecessary_column")
df.select("user_id", "email", "amount")

# Aggregations
df.groupBy("region").agg(
    F.sum("revenue").alias("total_revenue"),
    F.avg("order_amount").alias("avg_order"),
    F.countDistinct("user_id").alias("unique_users")
)

# Joins
users_df.join(orders_df, on="user_id", how="left")
users_df.join(orders_df, users_df.id == orders_df.user_id, "inner")

# Window functions
window = Window.partitionBy("user_id").orderBy("event_time")
df.withColumn("prev_event", F.lag("event_type", 1).over(window))
df.withColumn("user_rank", F.rank().over(Window.partitionBy("region").orderBy(F.desc("revenue"))))

# Reading and writing
spark.read.csv("path/", header=True, inferSchema=True)
spark.read.json("path/")
spark.read.parquet("path/")
spark.read.format("delta").load("path/")

df.write.parquet("path/", mode="overwrite", partitionBy=["date", "region"])
df.write.format("delta").mode("append").save("path/")
```

### Spark Performance Optimization

```python
# 1. Partition appropriately
df.repartition(200)                    # Increase parallelism
df.coalesce(10)                        # Reduce partitions before write

# 2. Cache frequently accessed DataFrames
df.cache()                             # Stores in memory
df.persist(StorageLevel.MEMORY_AND_DISK)  # Spills to disk if needed

# 3. Broadcast small tables (avoids expensive shuffle join)
from pyspark.sql.functions import broadcast
result = large_df.join(broadcast(small_lookup_df), "id")

# 4. Avoid UDFs when possible (they're slow — Catalyst optimizer can't optimize them)
# Instead of a Python UDF, use built-in functions:
df.withColumn("upper_name", F.upper(F.col("name")))  # ✓ Fast
# vs
from pyspark.sql.functions import udf
upper_udf = udf(lambda x: x.upper())  # ✗ Much slower

# 5. Push predicates down (filter early)
df.filter("date >= '2024-01-01'").groupBy("region").agg(...)  # ✓ Filter first
df.groupBy("region").agg(...).filter("date >= '2024-01-01'")  # ✗ Filter after aggregation

# 6. Configure shuffle partitions
spark.conf.set("spark.sql.shuffle.partitions", "400")  # Match cluster size
```

### Spark Batch vs Streaming

**Batch (Spark SQL)**:
```python
# Process a day's worth of data
df = spark.read.parquet(f"s3://bucket/raw/events/date={yesterday}/")
result = df.groupBy("user_id").agg(...)
result.write.parquet(f"s3://bucket/processed/daily/{yesterday}/")
```

**Spark Structured Streaming**:
```python
# Read from Kafka in real-time
from pyspark.sql import functions as F

df_stream = spark.readStream \
    .format("kafka") \
    .option("kafka.bootstrap.servers", "localhost:9092") \
    .option("subscribe", "user-events") \
    .load()

# Parse JSON messages
events = df_stream \
    .select(F.from_json(F.col("value").cast("string"), schema).alias("data")) \
    .select("data.*")

# Windowed aggregation
windowed = events \
    .withWatermark("event_time", "10 minutes") \
    .groupBy(
        F.window(F.col("event_time"), "5 minutes"),
        F.col("event_type")
    ) \
    .count()

# Write to sink
query = windowed.writeStream \
    .outputMode("append") \
    .format("delta") \
    .option("checkpointLocation", "s3://bucket/checkpoints/") \
    .start("s3://bucket/streaming-results/")

query.awaitTermination()
```

### SparkSQL

SparkSQL lets you query DataFrames with familiar SQL syntax:

```python
# Register as temp view
df.createOrReplaceTempView("events")

# Run SQL
result = spark.sql("""
    SELECT
        user_id,
        DATE_TRUNC('day', event_time) AS event_date,
        COUNT(*) AS event_count,
        SUM(CASE WHEN event_type = 'purchase' THEN amount ELSE 0 END) AS revenue
    FROM events
    WHERE event_time >= '2024-01-01'
    GROUP BY 1, 2
    ORDER BY revenue DESC
    LIMIT 100
""")
```

### Machine Learning with Spark (MLlib)

```python
from pyspark.ml import Pipeline
from pyspark.ml.feature import StringIndexer, VectorAssembler
from pyspark.ml.classification import RandomForestClassifier
from pyspark.ml.evaluation import BinaryClassificationEvaluator

# Feature engineering
indexer = StringIndexer(inputCol="category", outputCol="category_index")
assembler = VectorAssembler(
    inputCols=["amount", "age", "category_index"],
    outputCol="features"
)
classifier = RandomForestClassifier(featuresCol="features", labelCol="is_fraud")

# Pipeline
pipeline = Pipeline(stages=[indexer, assembler, classifier])

# Train
model = pipeline.fit(training_data)

# Predict
predictions = model.transform(test_data)

# Evaluate
evaluator = BinaryClassificationEvaluator(labelCol="is_fraud")
auc = evaluator.evaluate(predictions)
print(f"AUC: {auc}")
```

---

## MapReduce — The Foundation

MapReduce is the original distributed processing paradigm, created by Google in 2004. Understanding it makes you a better Spark engineer because Spark is built on similar concepts.

### How MapReduce Works

```
Input Data (HDFS)
      ↓
[MAP phase] — Transforms input into key-value pairs (runs in parallel)
      ↓
[SHUFFLE] — Groups all values with the same key to the same reducer
      ↓
[REDUCE phase] — Aggregates values per key (runs in parallel)
      ↓
Output (HDFS)
```

### MapReduce Example: Calculate Daily Average Temperature

**Input**:
```
2024-01-01 08:00 → 22°C
2024-01-01 12:00 → 28°C
2024-01-01 18:00 → 24°C
2024-01-02 08:00 → 20°C
2024-01-02 12:00 → 25°C
```

**Map phase** (cut timestamp to date, emit date→temp):
```
2024-01-01 → 22
2024-01-01 → 28
2024-01-01 → 24
2024-01-02 → 20
2024-01-02 → 25
```

**Shuffle** (group by key):
```
2024-01-01 → [22, 28, 24]
2024-01-02 → [20, 25]
```

**Reduce** (calculate average):
```
2024-01-01 → 24.7°C
2024-01-02 → 22.5°C
```

### MapReduce Limitations

1. **Disk I/O between stages**: Map output written to disk, reduce reads from disk → slow
2. **Only two stages**: Complex analytics requires chaining multiple MR jobs → very complex to write
3. **No streaming**: MR jobs have startup overhead — not suitable for real-time

Spark solves all three problems.

---

## HDFS — Hadoop Distributed File System

HDFS is the distributed storage layer of Hadoop. Understanding it helps you work with Spark on Hadoop clusters.

### HDFS Architecture

```
┌──────────────────────────────────────────────────────────┐
│  NAMENODE (Master)                                        │
│  - Metadata only (no actual data)                         │
│  - File paths, block locations, permissions               │
│  - Single point of failure (use NameNode HA)             │
└──────────────────────────────────────────────────────────┘
              ↓ manages
┌──────────────────────────────────────────────────────────┐
│  DATANODES (Workers) — store actual blocks               │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐        │
│  │  DataNode 1│  │  DataNode 2│  │  DataNode 3│        │
│  │  Block A1  │  │  Block A1  │  │  Block A2  │        │
│  │  Block B2  │  │  Block A2  │  │  Block B1  │        │
│  │  Block C1  │  │  Block C2  │  │  Block C1  │        │
│  └────────────┘  └────────────┘  └────────────┘        │
└──────────────────────────────────────────────────────────┘
```

### HDFS Key Properties

- **Block-based**: Files split into 128MB blocks (configurable)
- **Replication**: Default 3x replication for fault tolerance
- **Hardware independent**: Spans multiple machines/racks
- **Write-once, read-many**: Designed for streaming reads, not random writes
- **Locality**: Spark runs computations on the machine where blocks are stored

### When to Use HDFS vs Cloud Storage

| Aspect | HDFS | Cloud Storage (S3/GCS) |
|--------|------|----------------------|
| Cost | Hardware + ops | Pay per GB/request |
| Performance | Fast (co-located with compute) | Network latency |
| Management | You manage it | Fully managed |
| Scalability | Add nodes manually | Virtually unlimited |
| Modern stacks | Being replaced by cloud | Standard in modern stacks |

---

## Apache Flink — True Stream Processing

Apache Flink is a distributed stream processing framework designed for true real-time processing with low latency and exactly-once guarantees.

### Flink vs Spark Streaming

| Feature | Spark Streaming | Apache Flink |
|---------|----------------|-------------|
| Processing model | Micro-batches (seconds) | True event-by-event (milliseconds) |
| Latency | Seconds | Milliseconds |
| State management | Limited | First-class citizen |
| Exactly-once | Complex | Built-in |
| Learning curve | Lower (if you know Spark) | Higher |
| Use cases | Batch + near-real-time | Real-time, event-driven |

### When to Use Flink

- Financial fraud detection (need sub-second decisions)
- IoT sensor data (continuous high-frequency streams)
- Real-time feature engineering for ML models
- Event-driven microservices
- Complex event processing (CEP)

### Basic Flink Pattern

```python
# PyFlink example
from pyflink.datastream import StreamExecutionEnvironment
from pyflink.common.typeinfo import Types

env = StreamExecutionEnvironment.get_execution_environment()
env.set_parallelism(4)

# Read from Kafka
source = KafkaSource.builder() \
    .set_bootstrap_servers("localhost:9092") \
    .set_topics("user-events") \
    .set_group_id("flink-consumer") \
    .set_value_only_deserializer(SimpleStringSchema()) \
    .build()

stream = env.from_source(source, WatermarkStrategy.no_watermarks(), "Kafka Source")

# Process: filter, transform
result = stream \
    .filter(lambda msg: '"event_type": "purchase"' in msg) \
    .map(lambda msg: process_purchase(msg))

# Write to sink
result.sink_to(kafka_sink)

env.execute("Purchase Event Processor")
```

---

## REST APIs — The Connect Layer

REST APIs are the primary way data enters your platform from external systems.

### API Design Principles

1. **Stateless**: Each request contains all needed info — no server-side session
2. **Resource-based**: URLs represent resources (`/users/123`, not `/getUser?id=123`)
3. **HTTP verbs**: GET (read), POST (create), PUT (update/replace), PATCH (partial update), DELETE

### Consuming APIs in Pipelines

```python
import requests
from time import sleep

class APIClient:
    def __init__(self, base_url, api_key, rate_limit=100):
        self.base_url = base_url
        self.session = requests.Session()
        self.session.headers.update({'Authorization': f'Bearer {api_key}'})
        self.rate_limit = rate_limit
        self.request_count = 0

    def get(self, endpoint, params=None, retry=3):
        """GET with retry logic and rate limiting."""
        for attempt in range(retry):
            try:
                response = self.session.get(
                    f"{self.base_url}/{endpoint}",
                    params=params,
                    timeout=30
                )
                response.raise_for_status()

                self.request_count += 1
                if self.request_count % self.rate_limit == 0:
                    sleep(1)  # Rate limit: 100 requests/second

                return response.json()

            except requests.exceptions.HTTPError as e:
                if e.response.status_code == 429:  # Rate limited
                    sleep(2 ** attempt)             # Exponential backoff
                    continue
                raise
            except requests.exceptions.ConnectionError:
                if attempt < retry - 1:
                    sleep(2 ** attempt)
                    continue
                raise

    def paginate(self, endpoint, params=None):
        """Fetch all pages of paginated API."""
        page = 1
        while True:
            params = {**(params or {}), 'page': page, 'per_page': 100}
            data = self.get(endpoint, params)
            if not data:
                break
            yield from data
            page += 1
```

### OAuth 2.0 Authentication

```python
import requests

class OAuthClient:
    def __init__(self, client_id, client_secret, token_url):
        self.client_id = client_id
        self.client_secret = client_secret
        self.token_url = token_url
        self._token = None

    def get_token(self):
        if self._token:
            return self._token

        response = requests.post(self.token_url, data={
            'grant_type': 'client_credentials',
            'client_id': self.client_id,
            'client_secret': self.client_secret
        })
        self._token = response.json()['access_token']
        return self._token

    def get(self, url):
        headers = {'Authorization': f'Bearer {self.get_token()}'}
        return requests.get(url, headers=headers)
```

---

## Apache NiFi — Data Flow Automation

Apache NiFi provides a visual, no-code interface to build data pipelines. It's excellent for:
- Ingesting data from hundreds of different sources
- Routing data based on content
- Quick integration without writing code
- IoT data collection

### NiFi Capabilities
- Read from: REST APIs, databases, Kafka, files, FTP/SFTP, Twitter, S3
- Transform: Filter, split, merge, format conversion
- Write to: Kafka, databases, HDFS, S3, Elasticsearch, REST APIs

### When to Use NiFi vs Code

| Use NiFi | Write Code |
|----------|-----------|
| Simple integrations | Complex business logic |
| Rapid prototyping | Version control critical |
| Non-engineers maintain it | High performance needed |
| 100s of data sources | Testing and CI/CD required |

---

## Machine Learning in Production

As a data engineer, your role in ML is making data available and keeping models running reliably.

### The ML Pipeline (Data Engineer's View)

```
[Training Data Pipeline]
Source Data → Data Engineering Platform → Feature Engineering → Training Dataset
                                                                      ↓
                                                               [Data Scientist]
                                                               Trains Model
                                                                      ↓
[Production Inference Pipeline]                               Approved Model
Live Data → Feature Store → Model Serving → Predictions → Application
```

### Data Engineer's ML Responsibilities

1. **Feature pipelines**: Compute features from raw data at scale (batch + real-time)
2. **Feature store**: Centralized store of features for training AND serving (same features, consistent)
3. **Training data pipelines**: Supply labeled datasets, handle sampling, class imbalance
4. **Model deployment**: Package model, expose as API, monitor performance
5. **Retraining pipelines**: Detect model drift, trigger retraining automatically

### Why Production ML Is Hard

- **Data drift**: The real-world data distribution changes over time, breaking the model
- **Consistency**: Training features must match serving features exactly (training-serving skew)
- **Retraining**: Models degrade — you need automated pipelines to retrain and redeploy
- **Monitoring**: Track both model performance (accuracy) and data quality (distribution)
- **Latency**: Real-time inference must be fast (< 100ms), but features may be slow to compute

### MLflow for Experiment Tracking

```python
import mlflow

with mlflow.start_run():
    # Log parameters
    mlflow.log_param("learning_rate", 0.01)
    mlflow.log_param("n_estimators", 100)

    # Train model
    model = train_model(X_train, y_train)

    # Log metrics
    mlflow.log_metric("accuracy", accuracy)
    mlflow.log_metric("auc", auc)

    # Log model
    mlflow.sklearn.log_model(model, "model")
```

### AWS SageMaker

SageMaker provides managed infrastructure for:
- Model training (managed compute)
- Model hosting (auto-scaling endpoints)
- Model monitoring (detect drift automatically)
- Batch transform (run predictions at scale)

---

*← [03 - Essential Skills](03-essential-skills.md) | [05 - Data Storage](05-data-storage.md) →*
