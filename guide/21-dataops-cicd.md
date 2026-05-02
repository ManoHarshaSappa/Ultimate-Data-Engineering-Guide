# 21 — DataOps & CI/CD for Data Pipelines

> **Guide by [Mano Harsha Sappa](https://www.linkedin.com/in/manoharshasappa/)** | [Portfolio](https://manoharshasappa.github.io/portfolio_ManoHarshaSappa/) | [Back to Index](../README.md)

---

## Contents

- [What is DataOps?](#what-is-dataops)
- [CI/CD for Data Pipelines](#cicd-for-data-pipelines)
- [Testing Data Pipelines](#testing-data-pipelines)
- [GitHub Actions for Data](#github-actions-for-data)
- [Docker & Containerization](#docker--containerization)
- [Infrastructure as Code with Terraform](#infrastructure-as-code-with-terraform)
- [Environments: Dev → Staging → Prod](#environments-dev--staging--prod)
- [Deployment Strategies](#deployment-strategies)
- [Secrets Management](#secrets-management)
- [DataOps Best Practices](#dataops-best-practices)

---

## What is DataOps?

**DataOps** applies DevOps principles to data pipelines. It's a methodology that improves the speed, quality, and reliability of data analytics by:

- Automating testing and deployment of data pipelines
- Using version control for all data assets (code, configs, schemas)
- Enabling fast iteration with confidence
- Making data quality a first-class citizen in the delivery process

### DataOps vs DevOps

| DevOps | DataOps |
|--------|---------|
| Code as the artifact | Data + Code as artifacts |
| Unit/integration tests | Data quality tests + pipeline tests |
| CI/CD for applications | CI/CD for data pipelines |
| Monitoring app health | Monitoring data health |
| Rollback code | Rollback data (backfill/restore) |

### The DataOps Lifecycle

```
Code (SQL, Python, YAML)
        ↓
[Git Push / PR]
        ↓
[CI: Lint + Unit Tests + Data Tests]
        ↓
[Code Review]
        ↓
[CD: Deploy to Staging]
        ↓
[Integration Tests in Staging]
        ↓
[CD: Deploy to Production]
        ↓
[Monitor & Alert]
        ↓ (issue detected)
[Rollback or Hotfix]
```

---

## CI/CD for Data Pipelines

A proper CI/CD pipeline for data work includes:

### What to Check in CI

```
On every Pull Request:
  1. Lint SQL (sqlfluff)
  2. Lint Python (ruff/flake8)
  3. Validate dbt models compile (dbt compile)
  4. Run unit tests (pytest)
  5. Run dbt tests against a dev schema
  6. Check for data contract changes
  7. Generate and store documentation
```

### What to Deploy in CD

```
On merge to main:
  1. Deploy Airflow DAGs to staging
  2. Deploy dbt models to staging schema
  3. Run integration tests in staging
  4. Promote to production (auto or manual gate)
  5. Run smoke tests in production
  6. Update data catalog metadata
```

---

## Testing Data Pipelines

### Test Pyramid for Data

```
                    ┌───────────────┐
                    │   E2E Tests   │  ← Full pipeline run
                    │   (slow, few) │
                 ┌──┴───────────────┴──┐
                 │  Integration Tests  │  ← DB queries, pipeline steps
                 │   (medium speed)    │
              ┌──┴─────────────────────┴──┐
              │      Unit Tests           │  ← Python functions, transforms
              │   (fast, many, isolated)  │
           ┌──┴───────────────────────────┴──┐
           │     Data Quality Tests          │  ← dbt tests, GX
           │ (every pipeline run, automated) │
           └─────────────────────────────────┘
```

### Unit Tests for Python Transformations

```python
# src/transformers/order_transformer.py
def calculate_order_metrics(df):
    """Calculate derived order metrics."""
    df["order_value_usd"] = df["amount_cents"] / 100
    df["is_high_value"] = df["order_value_usd"] > 500
    df["days_to_ship"] = (df["shipped_at"] - df["created_at"]).dt.days
    return df

# tests/test_order_transformer.py
import pandas as pd
import pytest
from src.transformers.order_transformer import calculate_order_metrics

def test_order_value_conversion():
    df = pd.DataFrame({"amount_cents": [1000, 5500, 99]})
    result = calculate_order_metrics(df)
    assert result["order_value_usd"].tolist() == [10.0, 55.0, 0.99]

def test_high_value_flag():
    df = pd.DataFrame({"amount_cents": [60000, 40000, 50001]})
    result = calculate_order_metrics(df)
    assert result["is_high_value"].tolist() == [True, False, True]

def test_empty_dataframe():
    df = pd.DataFrame({"amount_cents": []})
    result = calculate_order_metrics(df)
    assert len(result) == 0

def test_days_to_ship():
    df = pd.DataFrame({
        "created_at": [pd.Timestamp("2024-01-01")],
        "shipped_at": [pd.Timestamp("2024-01-04")],
        "amount_cents": [1000],
    })
    result = calculate_order_metrics(df)
    assert result["days_to_ship"].iloc[0] == 3
```

### Testing Airflow DAGs

```python
# tests/test_dags.py
import pytest
from airflow.models import DagBag

@pytest.fixture
def dagbag():
    return DagBag(dag_folder="./dags", include_examples=False)

def test_no_import_errors(dagbag):
    assert len(dagbag.import_errors) == 0, f"DAG import errors: {dagbag.import_errors}"

def test_all_dags_have_owners(dagbag):
    for dag_id, dag in dagbag.dags.items():
        assert dag.default_args.get("owner") not in [None, "airflow"], \
            f"DAG {dag_id} has no owner set"

def test_all_dags_have_retries(dagbag):
    for dag_id, dag in dagbag.dags.items():
        retries = dag.default_args.get("retries", 0)
        assert retries >= 1, f"DAG {dag_id} has no retries configured"

def test_orders_dag_has_required_tasks(dagbag):
    dag = dagbag.get_dag("orders_pipeline")
    assert dag is not None
    task_ids = [task.task_id for task in dag.tasks]
    assert "extract_orders" in task_ids
    assert "validate_data" in task_ids
    assert "load_to_warehouse" in task_ids
```

### Testing dbt Models

```python
# tests/test_dbt_models.py
import subprocess

def test_dbt_models_compile():
    result = subprocess.run(
        ["dbt", "compile", "--profiles-dir", ".", "--profile", "ci"],
        capture_output=True, text=True
    )
    assert result.returncode == 0, f"dbt compile failed:\n{result.stderr}"

def test_dbt_tests_pass():
    result = subprocess.run(
        ["dbt", "test", "--profiles-dir", ".", "--profile", "ci",
         "--select", "tag:critical"],
        capture_output=True, text=True
    )
    assert result.returncode == 0, f"dbt tests failed:\n{result.stdout}"
```

---

## GitHub Actions for Data

### Complete CI Workflow

```yaml
# .github/workflows/ci.yml
name: Data Pipeline CI

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  lint:
    name: Lint Code
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      - name: Install dependencies
        run: pip install ruff sqlfluff

      - name: Lint Python
        run: ruff check .

      - name: Lint SQL
        run: sqlfluff lint models/ --dialect snowflake

  test:
    name: Unit Tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Run unit tests
        run: pytest tests/unit/ -v --cov=src --cov-report=xml

      - name: Upload coverage
        uses: codecov/codecov-action@v3

  dbt-compile:
    name: dbt Compile Check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      - name: Install dbt
        run: pip install dbt-snowflake

      - name: dbt compile
        env:
          DBT_SNOWFLAKE_ACCOUNT: ${{ secrets.DBT_SNOWFLAKE_ACCOUNT }}
          DBT_SNOWFLAKE_USER: ${{ secrets.DBT_SNOWFLAKE_USER }}
          DBT_SNOWFLAKE_PASSWORD: ${{ secrets.DBT_SNOWFLAKE_PASSWORD }}
        run: dbt compile --target ci --profiles-dir .

  dbt-test:
    name: dbt Tests (CI Schema)
    needs: dbt-compile
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install dbt
        run: pip install dbt-snowflake

      - name: dbt run + test on CI schema
        env:
          DBT_SNOWFLAKE_ACCOUNT: ${{ secrets.DBT_SNOWFLAKE_ACCOUNT }}
          DBT_SNOWFLAKE_USER: ${{ secrets.DBT_SNOWFLAKE_USER }}
          DBT_SNOWFLAKE_PASSWORD: ${{ secrets.DBT_SNOWFLAKE_PASSWORD }}
          DBT_TARGET_SCHEMA: ci_pr_${{ github.event.pull_request.number }}
        run: |
          dbt run --target ci --select state:modified+
          dbt test --target ci --select state:modified+

      - name: Cleanup CI schema
        if: always()
        run: dbt run-operation drop_schema --args "{'schema': 'ci_pr_${{ github.event.pull_request.number }}'}"

  deploy-staging:
    name: Deploy to Staging
    needs: [lint, test, dbt-test]
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Deploy Airflow DAGs to Staging
        run: |
          aws s3 sync ./dags/ s3://my-airflow-staging/dags/
          aws s3 sync ./plugins/ s3://my-airflow-staging/plugins/

      - name: Deploy dbt to Staging
        env:
          DBT_SNOWFLAKE_PASSWORD: ${{ secrets.DBT_SNOWFLAKE_PASSWORD_STAGING }}
        run: dbt run --target staging --full-refresh --select tag:staging
```

### CD Workflow (Production Deploy)

```yaml
# .github/workflows/cd.yml
name: Deploy to Production

on:
  workflow_dispatch:
    inputs:
      confirm:
        description: "Type 'deploy' to confirm production deployment"
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    if: github.event.inputs.confirm == 'deploy'
    steps:
      - uses: actions/checkout@v4

      - name: Deploy to Production
        run: |
          # Deploy Airflow DAGs
          aws s3 sync ./dags/ s3://my-airflow-prod/dags/
          
          # Deploy dbt models
          dbt run --target prod --select state:modified+
          dbt test --target prod --select state:modified+

      - name: Notify Slack
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {"text": "✅ Production deployment complete by ${{ github.actor }}"}
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

---

## Docker & Containerization

### Dockerfile for a Python Data Pipeline

```dockerfile
FROM python:3.11-slim

# Install system dependencies
RUN apt-get update && apt-get install -y \
    gcc \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# Set working directory
WORKDIR /app

# Install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY src/ ./src/
COPY config/ ./config/

# Non-root user for security
RUN useradd -m appuser && chown -R appuser:appuser /app
USER appuser

ENTRYPOINT ["python", "-m", "src.pipeline"]
```

### Docker Compose for Local Development

```yaml
# docker-compose.yml
version: "3.8"

services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: analytics
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: secret
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init_scripts:/docker-entrypoint-initdb.d

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
    ports:
      - "9092:9092"

  airflow:
    image: apache/airflow:2.8.0
    environment:
      AIRFLOW__CORE__SQL_ALCHEMY_CONN: postgresql+psycopg2://admin:secret@postgres/airflow
      AIRFLOW__CORE__EXECUTOR: LocalExecutor
    volumes:
      - ./dags:/opt/airflow/dags
      - ./plugins:/opt/airflow/plugins
    ports:
      - "8080:8080"
    depends_on:
      - postgres

  clickhouse:
    image: clickhouse/clickhouse-server:24.1
    ports:
      - "8123:8123"
      - "9000:9000"

volumes:
  postgres_data:
```

---

## Infrastructure as Code with Terraform

### Terraform for Snowflake Resources

```hcl
# terraform/main.tf
terraform {
  required_providers {
    snowflake = {
      source  = "Snowflake-Labs/snowflake"
      version = "~> 0.87"
    }
  }
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "data-platform/terraform.tfstate"
    region = "us-east-1"
  }
}

# Create databases
resource "snowflake_database" "analytics" {
  name    = "ANALYTICS"
  comment = "Main analytics database"
  data_retention_time_in_days = 7
}

resource "snowflake_schema" "raw" {
  database = snowflake_database.analytics.name
  name     = "RAW"
}

resource "snowflake_schema" "staging" {
  database = snowflake_database.analytics.name
  name     = "STAGING"
}

resource "snowflake_schema" "marts" {
  database = snowflake_database.analytics.name
  name     = "MARTS"
}

# Warehouses
resource "snowflake_warehouse" "transform" {
  name           = "TRANSFORM_WH"
  warehouse_size = "MEDIUM"
  auto_suspend   = 60
  auto_resume    = true
}

resource "snowflake_warehouse" "reporting" {
  name           = "REPORTING_WH"
  warehouse_size = "SMALL"
  auto_suspend   = 300
  auto_resume    = true
}

# Roles
resource "snowflake_role" "analyst" {
  name    = "ANALYST"
  comment = "Read-only access to mart tables"
}

resource "snowflake_role" "data_engineer" {
  name    = "DATA_ENGINEER"
  comment = "Full access to data platform"
}

# Grants
resource "snowflake_grant_privileges_to_role" "analyst_read" {
  role_name  = snowflake_role.analyst.name
  privileges = ["SELECT"]
  on_schema_object {
    object_type = "TABLE"
    all {
      in_schema = "ANALYTICS.MARTS"
    }
  }
}
```

### Terraform for AWS Data Infrastructure

```hcl
# S3 data lake buckets
resource "aws_s3_bucket" "data_lake" {
  bucket = "company-data-lake-${var.environment}"
  tags = {
    Environment = var.environment
    Team        = "data-platform"
  }
}

resource "aws_s3_bucket_versioning" "data_lake" {
  bucket = aws_s3_bucket.data_lake.id
  versioning_configuration {
    status = "Enabled"
  }
}

# Glue catalog
resource "aws_glue_catalog_database" "analytics" {
  name        = "analytics_${var.environment}"
  description = "Analytics data lake catalog"
}

# MWAA (Managed Airflow)
resource "aws_mwaa_environment" "airflow" {
  name              = "data-platform-${var.environment}"
  airflow_version   = "2.8.1"
  environment_class = "mw1.medium"
  source_bucket_arn = aws_s3_bucket.airflow.arn
  dag_s3_path       = "dags/"
  execution_role_arn = aws_iam_role.mwaa.arn

  airflow_configuration_options = {
    "core.default_task_retries" = "2"
    "core.parallelism"          = "32"
  }
}
```

---

## Environments: Dev → Staging → Prod

### Environment Structure

```
DEV (Developer Local)
  → Individual developer's local machine or dev cloud account
  → No shared resources, sandbox everything
  → Use Docker Compose, dev databases

STAGING (Shared Pre-Prod)
  → Mirrors production architecture
  → Real-sized data (or representative subset)
  → Used for integration testing before prod deploy
  → Resets periodically (weekly clone from prod)

PRODUCTION
  → Real data, real users, real money
  → Changes only via automated CD, never manual
  → Full monitoring and alerting
  → Strict access controls
```

### dbt Multi-Environment Setup

```yaml
# profiles.yml
my_project:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: "{{ env_var('SNOWFLAKE_ACCOUNT') }}"
      user: "{{ env_var('SNOWFLAKE_USER') }}"
      schema: "dbt_{{ env_var('USER') }}"  # personal dev schema
      warehouse: DEV_WH

    ci:
      type: snowflake
      account: "{{ env_var('SNOWFLAKE_ACCOUNT') }}"
      user: ci_user
      schema: "ci_pr_{{ env_var('PR_NUMBER', 'manual') }}"
      warehouse: CI_WH

    staging:
      type: snowflake
      schema: STAGING
      warehouse: STAGING_WH

    prod:
      type: snowflake
      schema: MARTS
      warehouse: PROD_WH
```

---

## Deployment Strategies

### Blue-Green Deployment for dbt

```bash
# Deploy to "green" schema while "blue" serves traffic
dbt run --target prod-green --full-refresh --select tag:critical

# Run tests against green
dbt test --target prod-green --select tag:critical

# If tests pass, swap alias (Snowflake view pointing to green)
dbt run-operation swap_schema --args '{"blue": "marts", "green": "marts_green"}'

# Old blue becomes the new green for next deployment
```

### Canary Deployments for Airflow

```python
# Run new DAG version on 10% of data first
@dag(schedule_interval="@hourly")
def orders_pipeline_v2():
    @task
    def canary_gate():
        import random
        # Only process 10% of traffic in canary mode
        if os.environ.get("CANARY_MODE") == "true":
            return random.random() < 0.10
        return True
    ...
```

---

## Secrets Management

Never hardcode credentials. Use proper secrets management:

### AWS Secrets Manager

```python
import boto3
import json

def get_secret(secret_name, region="us-east-1"):
    client = boto3.client("secretsmanager", region_name=region)
    response = client.get_secret_value(SecretId=secret_name)
    return json.loads(response["SecretString"])

# Usage
db_creds = get_secret("data-platform/postgres/prod")
conn = psycopg2.connect(
    host=db_creds["host"],
    user=db_creds["username"],
    password=db_creds["password"],
    database=db_creds["dbname"]
)
```

### GitHub Actions Secrets

```yaml
- name: Deploy
  env:
    SNOWFLAKE_PASSWORD: ${{ secrets.SNOWFLAKE_PASSWORD }}
    AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
    AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

### Airflow Connections (UI or Code)

```python
from airflow.models import Connection
from airflow import settings

# Programmatically create connection (better: use Terraform or UI)
conn = Connection(
    conn_id="snowflake_prod",
    conn_type="snowflake",
    host="account.snowflakecomputing.com",
    login="service_user",
    password="...",  # pull from secrets manager
    schema="MARTS"
)
session = settings.Session()
session.add(conn)
session.commit()
```

---

## DataOps Best Practices

### The Golden Rules

**1. Everything in Git**
- DAGs, SQL, Python, Terraform, configs, dbt models
- If it's not in Git, it doesn't exist

**2. Never Touch Production Manually**
- All changes go through PR → CI → CD
- Manual changes = untracked state = future incidents

**3. Test Everything**
- Unit tests for Python transformations
- dbt tests for every model
- DAG structure tests (no import errors, owners, retries)

**4. Idempotent Pipelines**
- Running a pipeline twice should produce the same result
- Every job should have a `--date` parameter to reprocess specific windows

**5. Monitor Before Users Notice**
- Freshness alerts, volume anomaly detection
- Data quality scorecard reviewed daily

**6. Fast Feedback Loops**
- CI should complete in < 10 minutes
- Developers should know within minutes if their change broke something

### DataOps Checklist for Every PR

```
Before merging:
  ✅ Python code passes linting (ruff)
  ✅ SQL passes sqlfluff
  ✅ All unit tests pass
  ✅ dbt models compile without errors
  ✅ dbt tests pass on CI schema
  ✅ New models have descriptions in schema.yml
  ✅ Sensitive columns are tagged (PII)
  ✅ No hardcoded credentials
  ✅ Airflow DAG has retries configured
  ✅ Backfill strategy documented if schema changes
```

### Measuring DataOps Maturity

```
Level 1 — Basics
  → Code in Git ✓
  → At least some automated tests ✓
  → Deployments are semi-manual

Level 2 — CI/CD
  → All code in Git ✓
  → CI runs on every PR ✓
  → Automated staging deployment ✓
  → dbt tests as quality gate ✓

Level 3 — Full DataOps
  → Full test pyramid ✓
  → Automated prod deployment ✓
  → Data quality monitoring ✓
  → Lineage tracked ✓
  → Data contracts enforced ✓
  → Incident process established ✓
```

---

*Previous: [20 — Real-Time Analytics](20-real-time-analytics.md) | Next: [22 — Salary & Career Guide](22-salary-career.md) →*
