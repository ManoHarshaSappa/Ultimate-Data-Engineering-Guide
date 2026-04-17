# 06 — Orchestration & Pipeline Management

> **Guide by [Mano Harsha Sappa](https://www.linkedin.com/in/manoharshasappa/)** | [Portfolio](https://manoharshasappa.github.io/portfolio_ManoHarshaSappa/) | [Back to Index](../README.md)

---

## Contents

- [Why Orchestration Matters](#why-orchestration-matters)
- [Apache Airflow](#apache-airflow)
- [Prefect](#prefect)
- [Kestra](#kestra)
- [Dagster](#dagster)
- [dbt — Data Build Tool](#dbt--data-build-tool)
- [CI/CD for Data Pipelines](#cicd-for-data-pipelines)
- [Choosing an Orchestrator](#choosing-an-orchestrator)

---

## Why Orchestration Matters

Without orchestration, you have Python scripts triggered manually or by cron. That breaks down when:

- **Dependencies**: Pipeline B must run AFTER Pipeline A finishes
- **Failures**: What happens when a job fails? You need retry logic and alerts
- **Visibility**: Which jobs ran? Which failed? When?
- **Scaling**: Running dozens or hundreds of pipelines daily
- **Backfilling**: Re-running historical data through a fixed pipeline

An orchestrator handles all of this. It's the brain that coordinates all your data workflows.

---

## Apache Airflow

Apache Airflow is the most widely used workflow orchestration platform. Written in Python, it uses DAGs (Directed Acyclic Graphs) to define workflows.

### Core Concepts

**DAG (Directed Acyclic Graph)**: A workflow definition — a collection of tasks with their dependencies.

**Task**: A single unit of work within a DAG.

**Operator**: A template that defines what a task does:
- `PythonOperator`: Run a Python function
- `BashOperator`: Run a bash command
- `PostgresOperator`: Run SQL on PostgreSQL
- `S3CopyObjectOperator`: Copy files in S3
- `BigQueryInsertJobOperator`: Run BigQuery job
- `SparkSubmitOperator`: Submit a Spark job

**Hook**: A connection to an external service (PostgreSQL, AWS, GCP, Kafka).

**Sensor**: A special operator that waits for a condition (file arrives, API endpoint returns 200, etc.).

**Connections**: Stored credentials for external services (managed in Airflow UI).

### Writing Your First DAG

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.operators.bash import BashOperator
from airflow.providers.postgres.operators.postgres import PostgresOperator
from airflow.providers.amazon.aws.operators.s3 import S3CopyObjectOperator
from airflow.sensors.filesystem import FileSensor
from datetime import datetime, timedelta

# Default arguments for all tasks
default_args = {
    'owner': 'mano_harsha',
    'depends_on_past': False,
    'start_date': datetime(2024, 1, 1),
    'email_on_failure': True,
    'email_on_retry': False,
    'email': ['sappamanoharsha@gmail.com'],
    'retries': 3,
    'retry_delay': timedelta(minutes=5),
    'retry_exponential_backoff': True,
}

# Define the DAG
with DAG(
    dag_id='daily_sales_pipeline',
    default_args=default_args,
    description='Daily sales data pipeline',
    schedule_interval='0 6 * * *',     # Every day at 6am
    catchup=False,                      # Don't backfill historical runs
    max_active_runs=1,                  # Only one run at a time
    tags=['sales', 'daily'],
) as dag:

    # Task 1: Extract data from source DB
    extract_sales = PostgresOperator(
        task_id='extract_sales',
        postgres_conn_id='source_postgres',
        sql="""
            COPY (
                SELECT * FROM sales
                WHERE DATE(created_at) = '{{ ds }}'
            ) TO '/tmp/sales_{{ ds }}.csv' CSV HEADER;
        """,
    )

    # Task 2: Upload to S3
    upload_to_s3 = BashOperator(
        task_id='upload_to_s3',
        bash_command='aws s3 cp /tmp/sales_{{ ds }}.csv s3://my-bucket/raw/sales/{{ ds }}/sales.csv',
    )

    # Task 3: Transform with Python
    def transform_data(ds, **kwargs):
        import pandas as pd
        import boto3

        # Read raw data
        s3 = boto3.client('s3')
        df = pd.read_csv(f's3://my-bucket/raw/sales/{ds}/sales.csv')

        # Transform
        df['revenue'] = df['quantity'] * df['unit_price']
        df['month'] = pd.to_datetime(df['sale_date']).dt.to_period('M').astype(str)

        # Write processed data
        output_path = f's3://my-bucket/processed/sales/{ds}/sales.parquet'
        df.to_parquet(output_path)

        # Pass data to next task via XCom
        return {'record_count': len(df), 'total_revenue': float(df['revenue'].sum())}

    transform_sales = PythonOperator(
        task_id='transform_sales',
        python_callable=transform_data,
        op_kwargs={'ds': '{{ ds }}'},
    )

    # Task 4: Load into warehouse
    load_to_warehouse = PostgresOperator(
        task_id='load_to_warehouse',
        postgres_conn_id='warehouse_postgres',
        sql="""
            INSERT INTO analytics.daily_sales
            SELECT * FROM staging.sales_{{ ds_nodash }}
            ON CONFLICT (sale_date) DO UPDATE
            SET revenue = EXCLUDED.revenue,
                record_count = EXCLUDED.record_count;
        """,
    )

    # Task 5: Run dbt models
    run_dbt = BashOperator(
        task_id='run_dbt_models',
        bash_command='cd /dbt && dbt run --select tag:daily --vars \'{"run_date": "{{ ds }}"}\' --profiles-dir /dbt',
    )

    # Task 6: Data quality check
    def check_data_quality(ti, ds):
        metrics = ti.xcom_pull(task_ids='transform_sales')
        record_count = metrics['record_count']
        total_revenue = metrics['total_revenue']

        if record_count < 100:
            raise ValueError(f"Too few records: {record_count}. Expected at least 100.")
        if total_revenue <= 0:
            raise ValueError(f"Revenue is non-positive: {total_revenue}")

        print(f"Quality check passed: {record_count} records, ${total_revenue:.2f} revenue")

    quality_check = PythonOperator(
        task_id='quality_check',
        python_callable=check_data_quality,
    )

    # Define dependencies (task flow)
    extract_sales >> upload_to_s3 >> transform_sales >> load_to_warehouse >> run_dbt >> quality_check
```

### Advanced Airflow Patterns

#### Dynamic DAG Generation
```python
# Generate one DAG per data source
sources = ['salesforce', 'hubspot', 'stripe', 'shopify']

for source in sources:
    with DAG(
        dag_id=f'ingest_{source}',
        schedule_interval='@daily',
        default_args=default_args,
    ) as dag:
        ingest = PythonOperator(
            task_id=f'ingest_{source}_data',
            python_callable=ingest_source,
            op_kwargs={'source': source},
        )
        globals()[f'dag_{source}'] = dag
```

#### Sensors
```python
from airflow.sensors.s3_key_sensor import S3KeySensor
from airflow.sensors.external_task import ExternalTaskSensor

# Wait for a file to appear in S3
wait_for_file = S3KeySensor(
    task_id='wait_for_source_file',
    bucket_name='my-bucket',
    bucket_key='raw/orders/{{ ds }}/orders.csv',
    aws_conn_id='aws_default',
    poke_interval=300,      # Check every 5 minutes
    timeout=7200,           # Fail after 2 hours
    mode='reschedule',      # Release worker slot while waiting
)

# Wait for another DAG's task to complete
wait_for_upstream = ExternalTaskSensor(
    task_id='wait_for_upstream_dag',
    external_dag_id='upstream_data_pipeline',
    external_task_id='transform_complete',
    execution_delta=timedelta(hours=1),
)
```

#### Branching
```python
from airflow.operators.python import BranchPythonOperator

def choose_path(**kwargs):
    day = kwargs['execution_date'].weekday()
    if day == 0:  # Monday
        return 'run_weekly_report'
    else:
        return 'run_daily_report'

branch = BranchPythonOperator(
    task_id='branching',
    python_callable=choose_path,
)
```

### Airflow Best Practices

1. **Idempotent tasks**: Running a task twice should produce the same result
2. **Atomic tasks**: Each task should be a single unit of work — not a giant script
3. **Use XCom sparingly**: Only for small metadata (not DataFrames!)
4. **Avoid top-level code in DAG files**: Database calls, API calls in DAG definitions = slow scheduler
5. **Use Connections**: Never hardcode credentials in DAGs
6. **Set timeouts**: Every task should have an `execution_timeout`
7. **Monitor lag**: Airflow UI shows task duration trends

---

## Prefect

Prefect is a modern alternative to Airflow with Python-native workflows and better developer experience.

### Key Differences from Airflow

| Feature | Airflow | Prefect |
|---------|---------|---------|
| Definition | DAG objects | Python functions |
| Deployment | Complex setup | Simple, cloud-native |
| Dynamic workflows | Possible but complex | Native |
| Local development | Needs Docker/server | Run locally instantly |
| Observability | Basic | Rich, first-class |

### Basic Prefect Flow

```python
from prefect import flow, task
from prefect.deployments import Deployment
from datetime import timedelta

@task(retries=3, retry_delay_seconds=60, log_prints=True)
def extract_data(source: str, date: str) -> list:
    """Extract data from source."""
    print(f"Extracting {source} data for {date}")
    # ... extraction logic
    return records

@task(retries=2)
def transform_data(records: list) -> list:
    """Clean and transform records."""
    return [clean_record(r) for r in records]

@task
def load_data(records: list, destination: str) -> int:
    """Load to destination."""
    # ... load logic
    return len(records)

@task
def send_alert(message: str):
    """Send Slack alert."""
    # ... slack webhook

@flow(name="Daily Sales Pipeline", log_prints=True)
def daily_sales_pipeline(date: str = None):
    """Main pipeline flow."""
    import pendulum
    date = date or pendulum.today().to_date_string()

    try:
        # Parallel extraction
        salesforce_future = extract_data.submit("salesforce", date)
        stripe_future = extract_data.submit("stripe", date)

        # Wait for both
        salesforce_records = salesforce_future.result()
        stripe_records = stripe_future.result()

        # Combine and transform
        all_records = salesforce_records + stripe_records
        cleaned_records = transform_data(all_records)

        # Load
        count = load_data(cleaned_records, "snowflake")
        print(f"Loaded {count} records")

    except Exception as e:
        send_alert(f"Pipeline failed: {e}")
        raise

if __name__ == "__main__":
    daily_sales_pipeline()
```

---

## Kestra

Kestra is a declarative, open-source orchestration platform. Workflows are defined in YAML, making it accessible to non-Python engineers.

### Basic Kestra Flow

```yaml
id: daily-sales-pipeline
namespace: data.analytics

triggers:
  - id: daily-schedule
    type: io.kestra.core.models.triggers.types.Schedule
    cron: "0 6 * * *"

tasks:
  - id: extract-from-postgres
    type: io.kestra.plugin.jdbc.postgresql.Query
    url: "{{ secret('POSTGRES_URL') }}"
    sql: |
      SELECT * FROM sales
      WHERE DATE(created_at) = '{{ execution.startDate | date("yyyy-MM-dd") }}'

  - id: upload-to-s3
    type: io.kestra.plugin.aws.s3.Upload
    accessKeyId: "{{ secret('AWS_ACCESS_KEY_ID') }}"
    secretKeyId: "{{ secret('AWS_SECRET_KEY') }}"
    region: us-east-1
    bucket: my-data-bucket
    key: "raw/sales/{{ execution.startDate | date('yyyy/MM/dd') }}/sales.csv"
    from: "{{ outputs['extract-from-postgres'].uri }}"

  - id: run-dbt
    type: io.kestra.plugin.dbt.cli.DbtCLI
    runner: DOCKER
    docker:
      image: ghcr.io/kestra-io/dbt-bigquery:latest
    commands:
      - dbt run --select tag:daily

  - id: notify-slack
    type: io.kestra.plugin.notifications.slack.SlackIncomingWebhook
    url: "{{ secret('SLACK_WEBHOOK') }}"
    payload: |
      {"text": "Daily sales pipeline completed successfully!"}

errors:
  - id: notify-failure
    type: io.kestra.plugin.notifications.slack.SlackIncomingWebhook
    url: "{{ secret('SLACK_WEBHOOK') }}"
    payload: |
      {"text": "ALERT: Daily sales pipeline failed! Error: {{ errorLogs() }}"}
```

---

## Dagster

Dagster takes a software-engineering-first approach to data orchestration. It focuses on data assets (not just tasks).

### Key Dagster Concepts

- **Asset**: A persistent artifact (table, file, ML model) produced by a pipeline
- **Op**: A single computation unit
- **Job**: A collection of Ops
- **Resource**: External dependencies (database connection, API client)
- **Schedule/Sensor**: How jobs are triggered

```python
from dagster import asset, op, job, resource, schedule
import pandas as pd

# Define assets (data-centric approach)
@asset(
    description="Raw sales data from Salesforce",
    group_name="raw",
    tags={"layer": "bronze"}
)
def raw_salesforce_data():
    """Extract sales data from Salesforce CRM."""
    # ... extraction logic
    return pd.DataFrame(...)

@asset(
    ins={"raw_data": AssetIn("raw_salesforce_data")},
    description="Cleaned sales data",
    group_name="transformed",
    tags={"layer": "silver"}
)
def cleaned_sales_data(raw_data: pd.DataFrame) -> pd.DataFrame:
    """Clean and validate sales data."""
    df = raw_data.dropna(subset=['order_id'])
    df['revenue'] = df['quantity'] * df['unit_price']
    return df

@asset(
    ins={"cleaned_data": AssetIn("cleaned_sales_data")},
    description="Daily revenue metrics",
    group_name="metrics",
    tags={"layer": "gold"}
)
def daily_revenue_metrics(cleaned_data: pd.DataFrame) -> pd.DataFrame:
    """Aggregate daily revenue metrics."""
    return cleaned_data.groupby('sale_date').agg(
        total_revenue=('revenue', 'sum'),
        order_count=('order_id', 'count')
    ).reset_index()
```

---

## dbt — Data Build Tool

dbt is the standard tool for data transformation in the modern data stack. It lets data engineers and analysts write transformations as SQL, with version control, testing, and documentation built in.

### Why dbt?

Before dbt, SQL transformations were:
- Scattered across dozens of stored procedures, views, and ETL scripts
- Not version-controlled
- Not documented
- Not tested
- Hard to understand and maintain

dbt fixes all of this.

### dbt Concepts

**Model**: A SQL file that defines a transformation. Each model becomes a table or view in your warehouse.

**Source**: A raw table in your warehouse that dbt reads from (but doesn't own).

**Materialization**:
- `view`: Run SQL each time someone queries it
- `table`: Materialize results as a table (refresh on each run)
- `incremental`: Append only new rows (fast, scalable)
- `ephemeral`: CTEs that only exist during a run (not persisted)

**Test**: Assertions about your data:
- `not_null`: Column has no nulls
- `unique`: Column values are unique
- `accepted_values`: Column only contains specific values
- `relationships`: Foreign key integrity

**Documentation**: Auto-generated documentation from your SQL and YAML descriptions.

### Project Structure

```
my_dbt_project/
├── dbt_project.yml            # Project configuration
├── profiles.yml               # Database connections
├── models/
│   ├── staging/               # Silver layer: clean raw data
│   │   ├── _sources.yml       # Define raw tables
│   │   ├── _schema.yml        # Tests + docs for staging models
│   │   ├── stg_salesforce__contacts.sql
│   │   ├── stg_stripe__payments.sql
│   │   └── stg_shopify__orders.sql
│   ├── intermediate/          # Complex transformations
│   │   └── int_orders_joined.sql
│   └── marts/                 # Gold layer: business aggregations
│       ├── finance/
│       │   ├── _schema.yml
│       │   └── fct_daily_revenue.sql
│       └── marketing/
│           └── dim_customers.sql
├── tests/                     # Custom tests
├── macros/                    # Reusable SQL functions
├── seeds/                     # Static CSV data
└── snapshots/                 # SCD Type 2 tracking
```

### Writing dbt Models

```sql
-- models/staging/stg_stripe__payments.sql
-- Staging: clean and rename columns from raw source

WITH source AS (
    SELECT * FROM {{ source('stripe', 'charges') }}
),

renamed AS (
    SELECT
        id                          AS payment_id,
        customer                    AS customer_id,
        amount / 100.0              AS amount,  -- Convert cents to dollars
        currency,
        status,
        LOWER(status)               AS status_normalized,
        TO_TIMESTAMP(created)       AS created_at,
        {{ dbt_utils.surrogate_key(['id', 'customer']) }} AS payment_sk
    FROM source
    WHERE status IN ('succeeded', 'pending', 'failed')
)

SELECT * FROM renamed
```

```sql
-- models/marts/finance/fct_daily_revenue.sql
-- Fact table: daily revenue metrics

{{ config(
    materialized='incremental',
    unique_key='sale_date',
    on_schema_change='append_new_columns'
) }}

WITH daily_payments AS (
    SELECT
        DATE(created_at) AS sale_date,
        SUM(amount)      AS gross_revenue,
        COUNT(*)         AS payment_count,
        COUNT(DISTINCT customer_id) AS unique_customers
    FROM {{ ref('stg_stripe__payments') }}
    WHERE status = 'succeeded'

    {% if is_incremental() %}
        AND created_at > (SELECT MAX(sale_date) FROM {{ this }})
    {% endif %}

    GROUP BY 1
)

SELECT
    sale_date,
    gross_revenue,
    payment_count,
    unique_customers,
    SUM(gross_revenue) OVER (ORDER BY sale_date) AS cumulative_revenue
FROM daily_payments
ORDER BY sale_date
```

### YAML Schema File (Tests + Documentation)

```yaml
# models/marts/finance/_schema.yml
version: 2

models:
  - name: fct_daily_revenue
    description: "Daily revenue metrics aggregated from Stripe payments"
    meta:
      owner: "Mano Harsha Sappa"
      domain: "Finance"
      tier: "Gold"

    columns:
      - name: sale_date
        description: "The date of the sale"
        tests:
          - not_null
          - unique

      - name: gross_revenue
        description: "Total revenue for the day (in USD)"
        tests:
          - not_null
          - dbt_utils.expression_is_true:
              expression: ">= 0"

      - name: payment_count
        description: "Number of successful payments"
        tests:
          - not_null

      - name: unique_customers
        description: "Number of distinct customers who paid"
        tests:
          - not_null
```

### dbt Commands

```bash
# Install dbt
pip install dbt-snowflake  # or dbt-bigquery, dbt-postgres, etc.

# Initialize project
dbt init my_project

# Run all models
dbt run

# Run specific models
dbt run --select stg_stripe__payments fct_daily_revenue
dbt run --select tag:daily           # All models tagged 'daily'
dbt run --select +fct_daily_revenue  # fct_daily_revenue and all its dependencies

# Test
dbt test
dbt test --select fct_daily_revenue

# Generate and serve documentation
dbt docs generate
dbt docs serve                        # Opens browser at localhost:8080

# Check for compilation errors
dbt compile

# Snapshot (SCD Type 2)
dbt snapshot

# Seed (load CSV files as tables)
dbt seed

# Fresh source check
dbt source freshness
```

### dbt Macros

```sql
-- macros/cents_to_dollars.sql
{% macro cents_to_dollars(column_name) %}
    ({{ column_name }} / 100.0)::DECIMAL(10,2)
{% endmacro %}

-- Usage in a model:
SELECT {{ cents_to_dollars('amount') }} AS amount_usd FROM raw_charges
```

---

## CI/CD for Data Pipelines

Treat your data pipeline code like production software: version control, testing, automated deployment.

### GitHub Actions Workflow for dbt

```yaml
# .github/workflows/dbt-ci.yml
name: dbt CI/CD

on:
  pull_request:
    branches: [main]
    paths: ['dbt/**']
  push:
    branches: [main]
    paths: ['dbt/**']

jobs:
  dbt-test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'

      - name: Install dbt
        run: pip install dbt-bigquery==1.7.0

      - name: Configure dbt profile
        run: |
          mkdir -p ~/.dbt
          echo "${{ secrets.DBT_PROFILES_YAML }}" > ~/.dbt/profiles.yml

      - name: dbt deps
        working-directory: ./dbt
        run: dbt deps

      - name: dbt compile
        working-directory: ./dbt
        run: dbt compile

      - name: dbt test (staging models only on PR)
        if: github.event_name == 'pull_request'
        working-directory: ./dbt
        run: dbt test --select staging

      - name: dbt run + test (all models on merge to main)
        if: github.event_name == 'push'
        working-directory: ./dbt
        run: |
          dbt run --select state:modified+ --defer --state prod-artifacts/
          dbt test --select state:modified+

      - name: Upload dbt artifacts
        if: github.event_name == 'push'
        uses: actions/upload-artifact@v3
        with:
          name: dbt-artifacts
          path: dbt/target/
```

---

## Choosing an Orchestrator

| Tool | Best For | Strengths | Weaknesses |
|------|---------|----------|------------|
| **Airflow** | Large teams, complex workflows | Mature, huge ecosystem, many operators | Heavy, complex setup |
| **Prefect** | Python-first teams | Great DX, easy local dev, cloud-native | Newer, smaller ecosystem |
| **Kestra** | Multi-language teams | YAML-based, easy for non-Python | Less mature |
| **Dagster** | Asset-centric platforms | Great observability, software-first | Steep learning curve |
| **dbt (alone)** | Analytics engineering | SQL-native, documentation, testing | Not a full orchestrator |

**Recommendation for most teams**:
- **Small team / startup**: Prefect Cloud or Kestra
- **Medium team**: Airflow (managed on Astronomer or GCC Composer) + dbt
- **Large enterprise**: Dagster or Airflow + dbt + Great Expectations

---

*← [05 - Data Storage](05-data-storage.md) | [07 - Cloud Platforms](07-cloud-platforms.md) →*
