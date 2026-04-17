# 07 — Cloud Platforms for Data Engineers

> **Guide by [Mano Harsha Sappa](https://www.linkedin.com/in/manoharshasappa/)** | [Portfolio](https://manoharshasappa.github.io/portfolio_ManoHarshaSappa/) | [Back to Index](../README.md)

---

## Contents

- [AWS for Data Engineers](#aws-for-data-engineers)
- [Google Cloud Platform (GCP)](#google-cloud-platform-gcp)
- [Microsoft Azure](#microsoft-azure)
- [Cloud vs On-Premises Decision Guide](#cloud-vs-on-premises-decision-guide)
- [Terraform — Infrastructure as Code](#terraform--infrastructure-as-code)
- [Cloud Cost Optimization](#cloud-cost-optimization)

---

## AWS for Data Engineers

AWS is the market leader with the broadest service portfolio. Most data engineering roles use AWS.

### Core AWS Data Services

#### Amazon S3 — Object Storage
The foundation of every AWS data platform. Virtually unlimited, cheap, durable storage.

```python
import boto3
import pandas as pd
from io import BytesIO

s3 = boto3.client('s3',
    region_name='us-east-1',
    aws_access_key_id='ACCESS_KEY',
    aws_secret_access_key='SECRET_KEY'
)

# Upload file
s3.upload_file('local_file.parquet', 'my-bucket', 'raw/data/2024/01/15/file.parquet')

# Upload DataFrame directly (no temp file needed)
df = pd.DataFrame({'a': [1, 2, 3]})
buffer = BytesIO()
df.to_parquet(buffer)
s3.put_object(Body=buffer.getvalue(), Bucket='my-bucket', Key='data/file.parquet')

# Download and read
response = s3.get_object(Bucket='my-bucket', Key='data/file.parquet')
df = pd.read_parquet(BytesIO(response['Body'].read()))

# List objects
paginator = s3.get_paginator('list_objects_v2')
for page in paginator.paginate(Bucket='my-bucket', Prefix='raw/orders/2024/'):
    for obj in page.get('Contents', []):
        print(obj['Key'], obj['Size'])

# S3 Select: query directly in S3 (avoid full download)
response = s3.select_object_content(
    Bucket='my-bucket',
    Key='data/large_file.csv',
    ExpressionType='SQL',
    Expression="SELECT * FROM S3Object WHERE amount > 100",
    InputSerialization={'CSV': {'FileHeaderInfo': 'USE'}},
    OutputSerialization={'CSV': {}}
)
```

**S3 Storage Classes** (cost optimization):
| Class | Use Case | Cost |
|-------|---------|------|
| S3 Standard | Frequently accessed | $$$ |
| S3 Standard-IA | Infrequently accessed (> 30 days) | $$ |
| S3 Glacier Instant | Archives, retrieved in ms | $ |
| S3 Glacier Flexible | Archives, 3-5 hours retrieval | ¢ |
| S3 Glacier Deep Archive | Long-term compliance | ¢¢ |

**S3 Lifecycle Rules**: Automatically move data to cheaper storage classes:
```json
{
  "Rules": [
    {
      "ID": "Move to IA after 30 days",
      "Transitions": [
        {"Days": 30, "StorageClass": "STANDARD_IA"},
        {"Days": 90, "StorageClass": "GLACIER"}
      ],
      "Expiration": {"Days": 365},
      "Status": "Enabled"
    }
  ]
}
```

#### Amazon Redshift — Data Warehouse

```sql
-- Create external schema (Redshift Spectrum — query S3 directly)
CREATE EXTERNAL SCHEMA s3_data
FROM DATA CATALOG DATABASE 'my_glue_database'
IAM_ROLE 'arn:aws:iam::ACCOUNT_ID:role/RedshiftRole';

-- Query data lake directly from Redshift
SELECT region, SUM(revenue) FROM s3_data.orders
WHERE year = '2024'
GROUP BY 1;

-- COPY from S3 (fastest way to load)
COPY orders
FROM 's3://my-bucket/processed/orders/'
IAM_ROLE 'arn:aws:iam::ACCOUNT_ID:role/RedshiftRole'
FORMAT AS PARQUET;

-- WLM (Workload Management) queues
-- Short queries get priority queue, long queries get separate queue

-- Vacuum and analyze
VACUUM orders;
ANALYZE orders;
```

#### AWS Glue — Managed ETL

Glue provides:
- **Glue Data Catalog**: Managed metadata store (tables, schemas, partitions)
- **Glue ETL**: Managed Spark jobs (PySpark)
- **Glue Crawlers**: Auto-discover schemas from S3/databases
- **Glue Studio**: Visual ETL builder

```python
# Glue ETL Job
import sys
from awsglue.transforms import *
from awsglue.utils import getResolvedOptions
from pyspark.context import SparkContext
from awsglue.context import GlueContext
from awsglue.job import Job

args = getResolvedOptions(sys.argv, ['JOB_NAME'])
sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext)
job.init(args['JOB_NAME'], args)

# Read from Glue Data Catalog
datasource = glueContext.create_dynamic_frame.from_catalog(
    database="raw_db",
    table_name="orders"
)

# Apply transforms
transformed = ApplyMapping.apply(
    frame=datasource,
    mappings=[
        ("order_id", "string", "order_id", "string"),
        ("amount", "double", "revenue_usd", "double"),
        ("created_date", "string", "sale_date", "date"),
    ]
)

# Write to S3
glueContext.write_dynamic_frame.from_options(
    frame=transformed,
    connection_type="s3",
    format="parquet",
    connection_options={
        "path": "s3://my-bucket/processed/orders/",
        "partitionKeys": ["year", "month"]
    }
)

job.commit()
```

#### Amazon EMR — Managed Spark/Hadoop

Run managed Spark clusters without managing servers.

```bash
# Create EMR cluster
aws emr create-cluster \
    --name "Data Pipeline Cluster" \
    --release-label emr-6.15.0 \
    --applications Name=Spark Name=Hive \
    --instance-type m5.xlarge \
    --instance-count 5 \
    --ec2-attributes KeyName=my-key-pair \
    --use-default-roles

# Submit Spark job
aws emr add-steps \
    --cluster-id j-XXXXXXXXXXXXX \
    --steps Type=Spark,Name="My Job",Args=[
        --deploy-mode,cluster,
        --class,com.example.Main,
        s3://my-bucket/jars/my-job.jar,
        s3://my-bucket/input/,
        s3://my-bucket/output/
    ]
```

#### AWS Kinesis — Real-Time Streaming

- **Kinesis Data Streams**: Real-time data streaming (like Kafka)
- **Kinesis Data Firehose**: Managed delivery to S3, Redshift, Elasticsearch
- **Kinesis Data Analytics**: Real-time SQL analytics on streams (Apache Flink)

```python
import boto3
import json

kinesis = boto3.client('kinesis', region_name='us-east-1')

# Produce record
kinesis.put_record(
    StreamName='user-events',
    Data=json.dumps({'user_id': '123', 'event': 'purchase', 'amount': 49.99}),
    PartitionKey='user_123'      # Determines shard
)

# Produce multiple records (more efficient)
kinesis.put_records(
    StreamName='user-events',
    Records=[
        {'Data': json.dumps(event), 'PartitionKey': event['user_id']}
        for event in events
    ]
)
```

#### AWS Lambda — Serverless Functions

Lambda runs code in response to events without managing servers. Perfect for:
- Triggering pipelines when files land in S3
- Simple transformations without a cluster
- Glue between services

```python
import boto3
import pandas as pd
import json

def lambda_handler(event, context):
    """Triggered when file is uploaded to S3."""
    # Parse S3 event
    bucket = event['Records'][0]['s3']['bucket']['name']
    key = event['Records'][0]['s3']['object']['key']

    print(f"Processing: s3://{bucket}/{key}")

    # Read the file
    s3 = boto3.client('s3')
    response = s3.get_object(Bucket=bucket, Key=key)
    df = pd.read_csv(response['Body'])

    # Simple transformation
    df['processed_at'] = pd.Timestamp.now().isoformat()
    df_clean = df.dropna()

    # Write processed file
    output_key = key.replace('raw/', 'processed/')
    buffer = df_clean.to_parquet(index=False)
    s3.put_object(Body=buffer, Bucket=bucket, Key=output_key)

    return {'statusCode': 200, 'records_processed': len(df_clean)}
```

#### Amazon Athena — Serverless SQL on S3

Query data in S3 with SQL. Pay per query. No cluster management.

```sql
-- Query Parquet files on S3
SELECT
    year,
    month,
    SUM(revenue) AS total_revenue,
    COUNT(*) AS order_count
FROM "raw_db"."orders"
WHERE year = '2024'
    AND month IN ('01', '02', '03')
GROUP BY 1, 2
ORDER BY 1, 2;

-- CTAS: Create table from query result
CREATE TABLE analytics.q1_summary
WITH (
    format = 'PARQUET',
    external_location = 's3://my-bucket/analytics/q1_summary/',
    partitioned_by = ARRAY['month']
) AS
SELECT * FROM raw_orders WHERE year = '2024' AND month <= '03';
```

#### AWS DMS — Database Migration Service

Migrates databases to AWS and supports ongoing replication (CDC):

```
Source DB (Oracle, SQL Server, MySQL, PostgreSQL)
         ↓ DMS replication instance
Target (Redshift, S3, RDS, DynamoDB)
```

Use for: Moving on-prem databases to cloud, setting up CDC pipelines.

### AWS Data Engineering Reference Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│                          AWS DATA PLATFORM                         │
│                                                                    │
│  Sources         Ingest           Store            Consume         │
│                                                                    │
│  RDS/Aurora ──→ DMS ──────────→                                   │
│  APIs ────────→ Lambda ───────→ S3 Data Lake ──→ Athena ─────────│→BI
│  Kafka ───────→ Kinesis ──────→ (Parquet+Iceberg) → EMR/Spark ───│
│  Files ───────→ S3 Event ─────→                → Redshift ───────│
│  SaaS Apps ───→ Glue Crawlers→                  → SageMaker ─────│→ML
│                               ↓                                   │
│                          Glue Data Catalog (Metadata)              │
│                               ↓                                   │
│                          Step Functions / MWAA (Airflow)           │
└────────────────────────────────────────────────────────────────────┘
```

---

## Google Cloud Platform (GCP)

GCP is renowned for its data analytics services — especially BigQuery, which is the gold standard for serverless data warehousing.

### Core GCP Data Services

#### BigQuery — The Best Serverless Data Warehouse

BigQuery is Google's flagship analytics service. No infrastructure to manage, auto-scales to petabytes.

```python
from google.cloud import bigquery

client = bigquery.Client(project='my-project')

# Run query
query = """
    SELECT
        DATE(event_timestamp) AS event_date,
        event_type,
        COUNT(*) AS event_count,
        SUM(revenue) AS total_revenue
    FROM `my-project.analytics.events`
    WHERE DATE(event_timestamp) = CURRENT_DATE() - 1
    GROUP BY 1, 2
"""

job = client.query(query)
results = job.result()

for row in results:
    print(f"{row.event_date}: {row.event_type} - {row.event_count}")

# Load from DataFrame
from google.cloud.bigquery import LoadJobConfig, WriteDisposition

job_config = LoadJobConfig(
    write_disposition=WriteDisposition.WRITE_APPEND,
    schema_update_options=['ALLOW_FIELD_ADDITION'],
    time_partitioning=bigquery.TimePartitioning(field='created_at')
)

df = pd.DataFrame(...)
client.load_table_from_dataframe(df, 'my-project.dataset.table', job_config=job_config)
```

**BigQuery features unique in the market**:
- **Serverless**: No cluster sizing, auto-scales, pay per query TB scanned
- **BigQuery ML**: Train ML models with SQL
- **BigQuery Omni**: Query data in AWS S3 or Azure from BigQuery
- **INFORMATION_SCHEMA**: Rich metadata about jobs, tables, usage

#### Cloud Storage (GCS)

Google's equivalent to AWS S3:

```python
from google.cloud import storage

client = storage.Client()
bucket = client.bucket('my-bucket')

# Upload file
blob = bucket.blob('raw/orders/2024/01/15/orders.parquet')
blob.upload_from_filename('local_orders.parquet')

# Download
blob.download_to_filename('local_copy.parquet')

# List objects
for blob in client.list_blobs('my-bucket', prefix='raw/orders/'):
    print(blob.name)
```

#### Cloud Dataflow — Managed Beam

Dataflow is Google's managed execution of Apache Beam pipelines (batch AND stream):

```python
import apache_beam as beam
from apache_beam.options.pipeline_options import PipelineOptions

options = PipelineOptions(
    runner='DataflowRunner',
    project='my-project',
    region='us-central1',
    temp_location='gs://my-bucket/temp/',
    job_name='daily-revenue-pipeline'
)

with beam.Pipeline(options=options) as p:
    (
        p
        | 'Read from GCS' >> beam.io.ReadFromText('gs://my-bucket/raw/orders/*.csv')
        | 'Parse CSV' >> beam.Map(parse_csv_row)
        | 'Filter valid' >> beam.Filter(lambda r: r['amount'] > 0)
        | 'Calculate revenue' >> beam.Map(calculate_daily_revenue)
        | 'Write to BigQuery' >> beam.io.WriteToBigQuery(
            'my-project:analytics.daily_revenue',
            write_disposition=beam.io.BigQueryDisposition.WRITE_APPEND
        )
    )
```

#### Cloud Pub/Sub — Managed Messaging

Google's equivalent to Kafka/Kinesis:

```python
from google.cloud import pubsub_v1

publisher = pubsub_v1.PublisherClient()
subscriber = pubsub_v1.SubscriberClient()

# Publish
topic_path = publisher.topic_path('my-project', 'user-events')
future = publisher.publish(
    topic_path,
    json.dumps({'user_id': '123', 'event': 'purchase'}).encode('utf-8'),
    event_type='purchase'  # Custom attribute
)

# Subscribe
subscription_path = subscriber.subscription_path('my-project', 'analytics-sub')

def callback(message):
    event = json.loads(message.data)
    process_event(event)
    message.ack()

subscriber.subscribe(subscription_path, callback=callback)
```

#### Dataproc — Managed Spark/Hadoop

```bash
# Create cluster
gcloud dataproc clusters create my-cluster \
    --region us-central1 \
    --zone us-central1-a \
    --master-machine-type n1-standard-4 \
    --worker-machine-type n1-standard-4 \
    --num-workers 4

# Submit Spark job
gcloud dataproc jobs submit pyspark \
    gs://my-bucket/scripts/pipeline.py \
    --cluster my-cluster \
    --region us-central1 \
    -- --date 2024-01-15 --output gs://my-bucket/processed/
```

---

## Microsoft Azure

Azure dominates enterprise data engineering, especially in organizations already using Microsoft products.

### Core Azure Data Services

#### Azure Data Lake Storage (ADLS Gen2)

```python
from azure.storage.filedatalake import DataLakeServiceClient

service_client = DataLakeServiceClient(
    account_url=f"https://{account_name}.dfs.core.windows.net",
    credential=credential
)

filesystem_client = service_client.get_file_system_client('my-datalake')
directory_client = filesystem_client.get_directory_client('raw/orders/2024/01/')
file_client = directory_client.create_file("orders.parquet")
file_client.upload_data(data, overwrite=True)
```

#### Azure Synapse Analytics

Unified analytics platform combining data warehouse (SQL Pools) + Spark + Pipelines:

```sql
-- Dedicated SQL Pool (data warehouse)
CREATE TABLE dbo.orders
WITH (
    DISTRIBUTION = HASH(user_id),    -- Distribute by user_id
    CLUSTERED COLUMNSTORE INDEX
)
AS SELECT * FROM OPENROWSET(
    BULK 'https://account.dfs.core.windows.net/datalake/raw/orders/',
    FORMAT='PARQUET'
) AS [result];

-- Serverless SQL Pool (query ADLS directly)
SELECT TOP 100 *
FROM OPENROWSET(
    BULK 'https://account.dfs.core.windows.net/datalake/raw/orders/**',
    FORMAT='PARQUET'
) AS data
WHERE data.year = '2024';
```

#### Azure Databricks

Fully managed Databricks platform on Azure:
- Collaborative notebooks (Python, Scala, SQL, R)
- Managed Spark clusters
- Delta Lake integration
- MLflow integration

#### Azure Data Factory

Visual ETL/ELT tool for orchestrating data movement:
- 90+ built-in connectors
- Data flows (visual Spark transformations)
- Integration with Azure DevOps for CI/CD

#### Azure Event Hubs

Azure's managed Kafka service. Kafka-compatible protocol.

```python
from azure.eventhub import EventHubProducerClient, EventData

producer = EventHubProducerClient.from_connection_string(
    conn_str="Endpoint=sb://...",
    eventhub_name="user-events"
)

with producer:
    event_data_batch = producer.create_batch()
    event_data_batch.add(EventData('{"user_id": "123", "event": "purchase"}'))
    producer.send_batch(event_data_batch)
```

---

## Cloud vs On-Premises Decision Guide

### When to Choose Cloud

1. **Variable workload**: Cloud scales elastically — you only pay for what you use
2. **Speed to market**: No hardware procurement, provision in minutes
3. **Global reach**: Deploy across regions without physical infrastructure
4. **Modern managed services**: Snowflake, BigQuery, Databricks are cloud-native
5. **Small-medium team**: No dedicated ops team needed

### When to Choose On-Premises

1. **Data sovereignty**: Regulatory requirements mandate local data storage
2. **Predictable large workload**: At very high utilization, owned hardware is cheaper
3. **Ultra-low latency**: Physical proximity to data sources
4. **Air-gapped security**: Some government/defense systems can't use cloud

### Hybrid Cloud

Most enterprises use hybrid: sensitive or high-volume data stays on-prem, burst workloads use cloud.

**Example**: A bank keeps customer PII data on-prem PostgreSQL, but replicates anonymized transaction data to BigQuery for analytics.

---

## Terraform — Infrastructure as Code

Terraform lets you define cloud infrastructure as code, enabling:
- **Reproducibility**: Same infra in dev, staging, prod
- **Version control**: Track infrastructure changes in Git
- **Automation**: Provision infrastructure in CI/CD pipelines
- **Multi-cloud**: Same tool for AWS, GCP, Azure

### Basic Terraform Concepts

```hcl
# main.tf — AWS S3 bucket + Glue database

terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "data-platform/terraform.tfstate"
    region = "us-east-1"
  }
}

provider "aws" {
  region = var.aws_region
}

# Variables
variable "aws_region" {
  default = "us-east-1"
}
variable "environment" {}

# S3 Data Lake bucket
resource "aws_s3_bucket" "data_lake" {
  bucket = "my-company-datalake-${var.environment}"
  tags = {
    Environment = var.environment
    Owner       = "data-engineering"
    Project     = "data-platform"
  }
}

resource "aws_s3_bucket_versioning" "data_lake" {
  bucket = aws_s3_bucket.data_lake.id
  versioning_configuration {
    status = "Enabled"
  }
}

# Glue Data Catalog
resource "aws_glue_catalog_database" "raw" {
  name = "raw_${var.environment}"
}

# Redshift cluster
resource "aws_redshift_cluster" "warehouse" {
  cluster_identifier = "my-warehouse-${var.environment}"
  database_name      = "analytics"
  master_username    = var.redshift_user
  master_password    = var.redshift_password
  node_type          = "ra3.xlplus"
  number_of_nodes    = 2

  skip_final_snapshot = var.environment != "prod"
}

# Outputs
output "data_lake_bucket" {
  value = aws_s3_bucket.data_lake.bucket
}

output "redshift_endpoint" {
  value = aws_redshift_cluster.warehouse.endpoint
}
```

```bash
# Terraform workflow
terraform init          # Download providers
terraform plan          # Show what will be created/changed
terraform apply         # Create/update infrastructure
terraform destroy       # Tear down all resources (careful!)

# Target specific resource
terraform apply -target=aws_s3_bucket.data_lake

# Workspaces for environments
terraform workspace new staging
terraform workspace select production
terraform apply -var-file=prod.tfvars
```

---

## Cloud Cost Optimization

Cloud costs can spiral out of control. Here are strategies to keep them in check.

### Big Data Cost Killers

1. **Leaving clusters running**: EMR/Dataproc clusters cost money every hour, even idle
2. **Oversized instances**: Using m5.4xlarge when m5.xlarge would suffice
3. **Too many table scans**: BigQuery charges per TB scanned — use partitioning!
4. **Storing everything in S3 Standard**: Move old data to Glacier
5. **Data transfer costs**: Moving data between regions is expensive

### Cost Control Strategies

**Rightsizing**: Match instance size to actual usage
```bash
# AWS Compute Optimizer recommendations
aws compute-optimizer get-ec2-instance-recommendations --region us-east-1
```

**Auto-scaling**: Scale up during peak hours, down during off-hours
```hcl
resource "aws_appautoscaling_target" "redshift" {
  min_capacity = 2
  max_capacity = 10
  # Scale based on CPU or storage
}
```

**Spot/Preemptible instances**: 60-90% savings for interruptible workloads (batch jobs)
```python
# EMR with Spot instances
instance_groups = [{
    'InstanceCount': 4,
    'InstanceRole': 'CORE',
    'InstanceType': 'm5.xlarge',
    'Market': 'SPOT',
    'BidPrice': '0.10'
}]
```

**BigQuery cost control**:
```sql
-- Preview query cost before running
-- Check "Bytes processed" estimate in UI

-- Partition pruning (HUGE cost savings)
SELECT * FROM `project.dataset.events`
WHERE DATE(_PARTITIONTIME) = '2024-01-15'  -- Only reads 1 day's partition

-- Use BI Engine for repeated dashboard queries
-- Reserve slots for predictable workloads
```

**Set budgets and alerts**:
```bash
# AWS Budgets
aws budgets create-budget \
    --account-id 123456789 \
    --budget Name=DataPlatformBudget,BudgetLimit={Amount=5000,Unit=USD},TimeUnit=MONTHLY

# GCP Budget with Pub/Sub alert
gcloud billing budgets create \
    --billing-account=BILLING_ACCOUNT_ID \
    --display-name="Data Platform Budget" \
    --budget-amount=5000USD \
    --threshold-rule=percent=80,basis=current-spend
```

---

*← [06 - Orchestration](06-orchestration.md) | [08 - dbt Modeling](08-dbt-modeling.md) →*
