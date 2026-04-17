# 03 — Essential Skills for Data Engineers

> **Guide by [Mano Harsha Sappa](https://www.linkedin.com/in/manoharshasappa/)** | [Portfolio](https://manoharshasappa.github.io/portfolio_ManoHarshaSappa/) | [Back to Index](../README.md)

---

## Contents

- [Python for Data Engineers](#python-for-data-engineers)
- [SQL — The Language of Data](#sql--the-language-of-data)
- [Git & Version Control](#git--version-control)
- [Linux Fundamentals](#linux-fundamentals)
- [Docker](#docker)
- [Cloud Fundamentals](#cloud-fundamentals)
- [Security & Privacy](#security--privacy)
- [Agile Development](#agile-development)
- [Computer Science Fundamentals](#computer-science-fundamentals)

---

## Python for Data Engineers

Python is the primary language of modern data engineering. It's versatile, readable, and has an enormous ecosystem of data libraries.

### Why Python?

- **Dominant in data**: pandas, PySpark, dbt, Airflow — all Python
- **Quick prototyping**: Ship pipeline logic fast
- **Community**: Most data engineering tutorials and libraries are Python-first
- **Versatility**: APIs, scripting, ML, web — Python does everything

### Essential Python Skills for DE

#### 1. Working with Files and Data
```python
import json
import csv
import pandas as pd

# Reading JSON
with open('data.json') as f:
    data = json.load(f)

# Reading CSV with pandas
df = pd.read_csv('data.csv')
df.head()

# Writing Parquet (column-optimized format)
df.to_parquet('output.parquet', engine='pyarrow', compression='snappy')
```

#### 2. Calling APIs
```python
import requests

# GET request
response = requests.get(
    'https://api.example.com/data',
    headers={'Authorization': 'Bearer TOKEN'},
    params={'start_date': '2024-01-01', 'limit': 1000}
)

data = response.json()

# Handling pagination
def fetch_all_pages(url, headers):
    all_data = []
    page = 1
    while True:
        r = requests.get(url, headers=headers, params={'page': page})
        batch = r.json()
        if not batch:
            break
        all_data.extend(batch)
        page += 1
    return all_data
```

#### 3. Database Connections
```python
from sqlalchemy import create_engine
import psycopg2

# SQLAlchemy (recommended)
engine = create_engine('postgresql://user:password@localhost:5432/mydb')
df = pd.read_sql('SELECT * FROM users WHERE created_at > %s', engine, params=['2024-01-01'])

# Write dataframe to DB
df.to_sql('processed_users', engine, if_exists='append', index=False)
```

#### 4. Data Cleaning (the real job)
```python
import pandas as pd

df = pd.read_csv('raw_data.csv')

# Handle missing values
df['age'].fillna(df['age'].median(), inplace=True)
df.dropna(subset=['user_id', 'email'], inplace=True)

# Remove duplicates
df.drop_duplicates(subset=['user_id'], keep='last', inplace=True)

# Type conversions
df['created_at'] = pd.to_datetime(df['created_at'])
df['revenue'] = df['revenue'].astype(float)

# String cleaning
df['email'] = df['email'].str.lower().str.strip()

# Validation
assert df['age'].between(0, 120).all(), "Invalid age values"
assert df['email'].str.contains('@').all(), "Invalid email addresses"
```

#### 5. Error Handling and Logging
```python
import logging

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler('pipeline.log'),
        logging.StreamHandler()
    ]
)
logger = logging.getLogger(__name__)

def process_batch(records):
    try:
        logger.info(f"Processing {len(records)} records")
        # ... processing logic
        logger.info("Batch processed successfully")
    except Exception as e:
        logger.error(f"Failed to process batch: {e}", exc_info=True)
        raise
```

#### 6. Environment Variables and Configuration
```python
import os
from dotenv import load_dotenv

load_dotenv()

DB_HOST = os.getenv('DB_HOST', 'localhost')
DB_PASSWORD = os.getenv('DB_PASSWORD')  # No default for secrets
API_KEY = os.getenv('API_KEY')

if not DB_PASSWORD:
    raise ValueError("DB_PASSWORD environment variable not set")
```

### Key Python Libraries for Data Engineers

| Library | Purpose |
|---------|---------|
| `pandas` | Data manipulation, cleaning |
| `pyarrow` | Fast Parquet/Feather I/O |
| `sqlalchemy` | Database connections |
| `requests` | HTTP API calls |
| `boto3` | AWS SDK |
| `google-cloud-*` | GCP SDKs |
| `pyspark` | Distributed data processing |
| `dlt` | Data load tool (modern ETL) |
| `great_expectations` | Data validation |
| `python-dotenv` | Environment management |

---

## SQL — The Language of Data

SQL is non-negotiable for data engineers. You need to know it deeply, not just at a basic level.

### Core SQL You Must Know

#### Window Functions (Critical for Analytics)
```sql
-- RANK, ROW_NUMBER, DENSE_RANK
SELECT
    user_id,
    order_amount,
    ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at DESC) AS order_rank,
    RANK() OVER (PARTITION BY region ORDER BY revenue DESC) AS regional_rank,
    DENSE_RANK() OVER (ORDER BY order_amount DESC) AS amount_rank
FROM orders;

-- LAG/LEAD for time-series comparisons
SELECT
    date,
    revenue,
    LAG(revenue, 1) OVER (ORDER BY date) AS prev_day_revenue,
    revenue - LAG(revenue, 1) OVER (ORDER BY date) AS day_over_day_change,
    LEAD(revenue, 7) OVER (ORDER BY date) AS next_week_revenue
FROM daily_metrics;

-- Running totals and moving averages
SELECT
    date,
    revenue,
    SUM(revenue) OVER (ORDER BY date ROWS UNBOUNDED PRECEDING) AS cumulative_revenue,
    AVG(revenue) OVER (ORDER BY date ROWS 6 PRECEDING) AS rolling_7day_avg
FROM daily_metrics;
```

#### CTEs (Common Table Expressions)
```sql
WITH
user_orders AS (
    SELECT
        user_id,
        COUNT(*) AS total_orders,
        SUM(amount) AS total_revenue
    FROM orders
    WHERE status = 'completed'
    GROUP BY user_id
),
user_segments AS (
    SELECT
        user_id,
        total_orders,
        total_revenue,
        CASE
            WHEN total_revenue > 1000 THEN 'High Value'
            WHEN total_revenue > 100 THEN 'Medium Value'
            ELSE 'Low Value'
        END AS segment
    FROM user_orders
)
SELECT
    u.name,
    u.email,
    s.segment,
    s.total_revenue
FROM users u
JOIN user_segments s ON u.id = s.user_id;
```

#### Slowly Changing Dimensions (SCD) in SQL
```sql
-- SCD Type 2: Track historical changes
SELECT
    product_id,
    price,
    valid_from,
    COALESCE(valid_to, '9999-12-31') AS valid_to,
    CASE WHEN valid_to IS NULL THEN TRUE ELSE FALSE END AS is_current
FROM product_history
WHERE product_id = 42;
```

#### Optimizing SQL Queries
```sql
-- Use EXPLAIN to understand query plan
EXPLAIN ANALYZE
SELECT * FROM large_table WHERE created_at > '2024-01-01';

-- Partition pruning: query only relevant partitions
SELECT * FROM events
WHERE event_date BETWEEN '2024-01-01' AND '2024-01-31'  -- Uses partition
AND event_type = 'purchase';

-- Avoid SELECT *, fetch only needed columns
SELECT user_id, total_amount FROM orders  -- ✓ Good
SELECT * FROM orders                       -- ✗ Fetches unnecessary columns
```

### SQL Interview Must-Know Topics
- INNER JOIN vs LEFT JOIN vs FULL OUTER JOIN vs CROSS JOIN
- Aggregations: GROUP BY, HAVING
- Window functions: RANK, ROW_NUMBER, LAG, LEAD, SUM/AVG OVER
- CTEs and subqueries
- UNION vs UNION ALL
- Indexes and when to use them
- Query optimization and EXPLAIN plans
- ACID properties
- Stored procedures and views

---

## Git & Version Control

Git is the backbone of professional software development. Without it, your pipeline code is chaos.

### Why Git Matters for Data Engineers

- Track every change to your pipeline code
- Roll back when something breaks production
- Collaborate with teammates without overwriting each other's work
- Document changes with meaningful commit messages
- CI/CD pipelines are triggered by Git events

### Core Git Commands

```bash
# Setup
git config --global user.name "Mano Harsha Sappa"
git config --global user.email "sappamanoharsha@gmail.com"

# Start a new repo
git init
git clone https://github.com/username/repo.git

# Day-to-day workflow
git status                          # See what's changed
git add specific_file.py            # Stage specific file (never git add .)
git add -p                          # Interactive staging (review each change)
git commit -m "feat: add retry logic to Kafka consumer"

# Branching
git checkout -b feature/new-pipeline  # Create and switch to new branch
git checkout main                     # Switch to main
git merge feature/new-pipeline        # Merge branch
git branch -d feature/new-pipeline    # Delete merged branch

# Syncing with remote
git pull origin main                  # Pull latest changes
git push origin feature/new-pipeline  # Push your branch

# Undoing mistakes
git restore file.py                   # Discard unstaged changes
git revert HEAD                       # Undo last commit (safe, creates new commit)
git stash                             # Temporarily save changes

# Inspecting
git log --oneline --graph             # Visual commit history
git diff HEAD~1                       # Diff with previous commit
git blame pipeline.py                 # Who wrote each line
```

### Git Best Practices for Data Engineers

1. **Never commit secrets** — Use `.gitignore` for `.env`, `credentials.json`, API keys
2. **Meaningful commit messages** — `fix: handle null values in user_id column` not `fix stuff`
3. **Small, focused commits** — One logical change per commit
4. **Branch per feature** — Never commit directly to `main`
5. **PR reviews** — Every change reviewed before merging
6. **Tag releases** — `git tag v1.2.0` before deploying

```gitignore
# .gitignore for data engineering projects
.env
*.pem
credentials.json
service_account.json
*.csv
*.parquet
*.log
__pycache__/
.venv/
dist/
*.egg-info/
```

---

## Linux Fundamentals

Most data engineering infrastructure runs on Linux. You need to be comfortable on the command line.

### Essential Commands

```bash
# Navigation
pwd                     # Print working directory
ls -la                  # List files with details and hidden files
cd /path/to/directory   # Change directory
cd ..                   # Go up one level

# Files
cat file.txt            # Display file content
head -n 50 large.csv    # First 50 lines
tail -f logs/app.log    # Follow log file in real-time
wc -l file.csv          # Count lines
grep "ERROR" app.log    # Find lines containing "ERROR"
grep -i "error" app.log # Case-insensitive search

# File operations
cp source.txt dest.txt
mv old_name.txt new_name.txt
rm file.txt
rm -rf directory/       # Recursive delete (be careful!)
mkdir -p path/to/dir    # Create directory with parents

# Permissions
chmod 755 script.sh     # Make executable
chown user:group file   # Change owner
ls -l                   # See permissions

# Process management
ps aux                  # List all processes
top                     # Real-time process monitor
htop                    # Better process monitor
kill -9 PID             # Force kill a process
nohup command &         # Run in background, persist after logout

# Disk and memory
df -h                   # Disk space
du -sh directory/       # Directory size
free -h                 # Memory usage

# Networking
curl -X GET https://api.example.com/data  # HTTP request
wget https://example.com/file.zip         # Download file
netstat -tulpn          # Show open ports
ssh user@server         # SSH into server
```

### Shell Scripting

```bash
#!/bin/bash
# Example: Daily data pipeline script

# Exit on any error
set -e

# Variables
DATE=$(date +%Y-%m-%d)
DATA_DIR="/data/raw/${DATE}"
LOG_FILE="/logs/pipeline_${DATE}.log"

# Functions
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

# Create directories
mkdir -p "$DATA_DIR"

log "Starting pipeline for $DATE"

# Download data
log "Downloading data..."
curl -s "https://api.example.com/data?date=${DATE}" -o "${DATA_DIR}/raw.json"

# Process data
log "Processing data..."
python3 /pipeline/process.py --input "${DATA_DIR}/raw.json" --output "${DATA_DIR}/processed.parquet"

log "Pipeline completed successfully"
```

### Cron Jobs — Scheduling Tasks

```bash
# Edit crontab
crontab -e

# Cron syntax: minute hour day month weekday command
# Examples:
0 6 * * *    /scripts/daily_pipeline.sh     # Every day at 6am
0 */4 * * *  /scripts/refresh_cache.sh      # Every 4 hours
30 23 * * 0  /scripts/weekly_report.sh      # Every Sunday at 11:30pm
*/5 * * * *  /scripts/health_check.sh       # Every 5 minutes

# IMPORTANT: Always end cron files with an empty line
```

---

## Docker

Docker containers are the standard way to package and run data engineering applications.

### Why Docker for Data Engineers?

- **Reproducibility**: "Works on my machine" becomes "works everywhere"
- **Isolation**: Run different versions of Python, Spark, etc. without conflicts
- **Portability**: Build once, run anywhere (local, server, cloud)
- **Microservices**: Run Kafka, PostgreSQL, Airflow all in containers

### Core Docker Concepts

| Concept | Description |
|---------|-------------|
| **Image** | Blueprint for a container (read-only) |
| **Container** | Running instance of an image |
| **Dockerfile** | Instructions to build an image |
| **Docker Hub** | Public registry of Docker images |
| **Volume** | Persistent storage for containers |
| **Network** | Communication between containers |

### Essential Docker Commands

```bash
# Images
docker pull postgres:15                  # Download image
docker images                            # List images
docker rmi image_name                    # Remove image
docker build -t my-pipeline:v1.0 .      # Build from Dockerfile

# Containers
docker run -d --name my-container \
    -p 5432:5432 \
    -e POSTGRES_PASSWORD=secret \
    -v /local/data:/var/lib/postgresql/data \
    postgres:15

docker ps                                # List running containers
docker ps -a                             # List all containers (including stopped)
docker stop my-container                 # Stop container
docker start my-container                # Start stopped container
docker rm my-container                   # Remove container
docker logs my-container                 # View logs
docker exec -it my-container bash        # Enter container shell
docker inspect my-container             # Container configuration

# Networks
docker network create my-network --driver bridge
docker network connect my-network my-container
docker network ls

# Cleanup
docker system prune                      # Remove all stopped containers, unused images
```

### Writing a Dockerfile

```dockerfile
# Dockerfile for a Python data pipeline
FROM python:3.11-slim

# Set working directory
WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    gcc \
    && rm -rf /var/lib/apt/lists/*

# Install Python dependencies first (better caching)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

# Set environment variables
ENV PYTHONUNBUFFERED=1

# Run the pipeline
CMD ["python", "pipeline.py"]
```

### Docker Compose — Multi-Container Setup

```yaml
# docker-compose.yml — Full data stack for development
version: '3.8'

services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: datawarehouse
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - data-network

  kafka:
    image: bitnami/kafka:latest
    depends_on:
      - zookeeper
    environment:
      KAFKA_CFG_ZOOKEEPER_CONNECT: zookeeper:2181
      ALLOW_PLAINTEXT_LISTENER: 'yes'
    ports:
      - "9092:9092"
    networks:
      - data-network

  zookeeper:
    image: bitnami/zookeeper:latest
    environment:
      ALLOW_ANONYMOUS_LOGIN: 'yes'
    networks:
      - data-network

  airflow-webserver:
    image: apache/airflow:2.8.0
    command: webserver
    ports:
      - "8080:8080"
    environment:
      AIRFLOW__CORE__EXECUTOR: LocalExecutor
      AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql+psycopg2://admin:secret@postgres/airflow
    depends_on:
      - postgres
    networks:
      - data-network

volumes:
  postgres_data:

networks:
  data-network:
    driver: bridge
```

```bash
docker-compose up -d          # Start all services
docker-compose down           # Stop all services
docker-compose logs -f kafka  # Follow Kafka logs
docker-compose ps             # Status of all services
```

### Kubernetes (Overview)

When you have multiple containers that need to scale and stay available, you use Kubernetes:

- **Pod**: One or more containers running together
- **Deployment**: Manages replicas of your pod
- **Service**: Exposes pods to network traffic
- **Node**: A machine in the cluster

Kubernetes lets you run your Spark workers, Kafka brokers, and Airflow on a cluster with automatic scaling, health checks, and load balancing. This is how companies like Netflix and Uber run their data infrastructure.

---

## Cloud Fundamentals

### IaaS vs PaaS vs SaaS

| Model | You Manage | Provider Manages | Examples |
|-------|-----------|-----------------|---------|
| **IaaS** (Infrastructure as a Service) | OS, runtime, apps | Hardware, network | AWS EC2, Google Compute Engine |
| **PaaS** (Platform as a Service) | Apps, data | OS, runtime, infrastructure | Heroku, Google App Engine |
| **SaaS** (Software as a Service) | Configuration only | Everything | Snowflake, Fivetran, Airbyte Cloud |

### Cloud vs On-Premises

**Cloud advantages**:
- Pay-as-you-go pricing (no upfront hardware cost)
- Elastic scaling (scale up in minutes, not months)
- Managed services (no ops overhead)
- Global availability

**On-premises advantages**:
- Full control over hardware and data
- Predictable costs at large scale
- Data sovereignty requirements (some governments require local storage)
- Lower latency for on-prem data sources

**Hybrid Cloud**: A mix of both — keep sensitive data on-premises, use cloud for elastic compute.

### The Three Major Cloud Providers

#### AWS (Amazon Web Services) — Market Leader
Key data services:
- **S3**: Object storage (data lake foundation)
- **Redshift**: Data warehouse
- **EMR**: Managed Hadoop/Spark
- **Kinesis**: Real-time streaming
- **Glue**: Managed ETL service
- **Athena**: SQL queries on S3
- **RDS**: Managed relational databases
- **DynamoDB**: NoSQL key-value store
- **Lambda**: Serverless compute

#### GCP (Google Cloud Platform) — Best for Analytics
Key data services:
- **BigQuery**: Best-in-class serverless data warehouse
- **Cloud Storage**: Object storage
- **Dataflow**: Apache Beam managed streaming/batch
- **Pub/Sub**: Managed message queue
- **Dataproc**: Managed Spark/Hadoop
- **Cloud SQL**: Managed relational databases
- **Bigtable**: NoSQL wide-column store
- **Looker**: BI tool (Google-owned)

#### Azure (Microsoft) — Enterprise Dominant
Key data services:
- **Azure Data Lake Storage**: Object storage
- **Azure Synapse Analytics**: Unified analytics platform
- **Azure Data Factory**: Managed ETL
- **Azure Event Hubs**: Managed Kafka
- **Azure Databricks**: Managed Spark
- **SQL Database**: Managed SQL Server
- **Cosmos DB**: Multi-model NoSQL
- **Power BI**: BI tool

---

## Security & Privacy

### SSL/TLS Certificates

Data in transit should always be encrypted using TLS (Transport Layer Security):
- HTTPS uses TLS to encrypt web traffic
- Kafka connections use TLS to protect messages
- Database connections should use SSL

### JSON Web Tokens (JWT)

JWTs are the standard for API authentication:
```json
{
  "header": { "alg": "RS256", "typ": "JWT" },
  "payload": {
    "user_id": "12345",
    "roles": ["data_engineer"],
    "exp": 1735689600
  },
  "signature": "..."
}
```

### GDPR Compliance

The EU's General Data Protection Regulation (GDPR) applies to any personal data of EU citizens:

**Key principles for data engineers**:
- **Right to Erasure**: Users can request their data be deleted — your pipelines must support this
- **Data Minimization**: Collect only what you need
- **Purpose Limitation**: Use data only for stated purposes
- **Privacy by Design**: Build privacy in from the start

**Practical implications**:
- PII (names, emails, IPs) must be encrypted or anonymized in data warehouses
- Data retention policies must be enforced automatically
- Access to personal data must be logged
- Data lineage must be tracked (where does PII flow?)

### Privacy by Design

Don't bolt on privacy as an afterthought. Build it in:
1. **Pseudonymization**: Replace real IDs with hashed/tokenized IDs
2. **Encryption at rest**: Encrypt sensitive columns in the warehouse
3. **Role-based access**: Only authorized users/roles see PII
4. **Data masking**: Mask PII in non-production environments
5. **Audit logging**: Track who accessed what data and when

---

## Agile Development

Data engineering teams work in Agile frameworks. Understanding this helps you succeed in team environments.

### Why Agile Matters

Traditional waterfall development (plan → build → test → release all at once) fails in data engineering because:
- Requirements change as data is explored
- Pipelines need to evolve with source systems
- Stakeholder needs shift constantly

Agile responds to change rather than following a fixed plan.

### The Agile Rules That Actually Work

1. **Self-reliance is king** — Keep critical knowledge in-house. Outsourcing core data infrastructure means losing the ability to quickly iterate.
2. **Deliver incrementally** — Ship a working v1 pipeline in week 1. Improve iteratively.
3. **Fail fast** — When something doesn't work, find out quickly. Don't invest months in the wrong direction.
4. **Data over opinions** — A data engineer's superpower: when in doubt, show the data.

### Scrum for Data Teams

**Sprint** (2 weeks):
- Sprint planning: choose tasks from backlog
- Daily standup: 15-min sync on progress/blockers
- Sprint review: demo what was built
- Retrospective: what went well, what to improve

**For data engineering**:
- Backlog items: "Build user events pipeline", "Add data quality tests to orders model"
- Metrics: pipeline reliability, data freshness SLA, coverage of data quality checks

### OKR Framework

**Objective + Key Results** — Great for small data teams:

Example:
- **Objective**: Make data platform reliable and trusted
  - **KR1**: Reduce pipeline failures from 15% to 3% this quarter
  - **KR2**: Add data quality checks covering 90% of critical tables
  - **KR3**: Achieve < 5 min data freshness SLA for 100% of dashboards

---

## Computer Science Fundamentals

You don't need a CS degree, but these fundamentals make you a better data engineer.

### Data Structures

| Structure | When to Use | DE Context |
|-----------|-----------|-----------|
| **Array/List** | Sequential data | DataFrame columns |
| **Hash Map/Dict** | Key-value lookups | Deduplication, joins |
| **Queue** | FIFO processing | Kafka consumers |
| **Set** | Uniqueness check | Finding distinct values |
| **Tree** | Hierarchical data | DAGs, expression trees |
| **Graph** | Relationships | Data lineage, dependency graphs |

### Algorithms

- **Sorting**: Know merge sort O(n log n) — how Spark sorts partitions
- **Hashing**: How distributed systems partition data across nodes
- **MapReduce** as an algorithm: Map (transform) + Reduce (aggregate)
- **Compression**: Why Parquet + Snappy is the standard for storage

### Networking Basics

**OSI Model** (how data flows through a network):
```
7. Application   → HTTP, gRPC, JDBC
6. Presentation  → SSL/TLS encryption
5. Session       → Connection management
4. Transport     → TCP/UDP
3. Network       → IP routing
2. Data Link     → Ethernet frames
1. Physical      → Cables, fiber
```

**For data engineers, most important**:
- **TCP vs UDP**: Kafka uses TCP (reliable delivery). UDP sometimes used for metrics.
- **HTTP/REST**: Standard for APIs
- **Ports**: PostgreSQL (5432), Kafka (9092), Airflow (8080), Redis (6379)

---

*← [02 - Fundamentals](02-fundamentals.md) | [04 - Advanced Skills](04-advanced-skills.md) →*
