# 18 — Data Observability & Data Quality

> **Guide by [Mano Harsha Sappa](https://www.linkedin.com/in/manoharshasappa/)** | [Portfolio](https://manoharshasappa.github.io/portfolio_ManoHarshaSappa/) | [Back to Index](../README.md)

---

## Contents

- [What is Data Observability?](#what-is-data-observability)
- [The Five Pillars of Data Observability](#the-five-pillars-of-data-observability)
- [Data Quality Dimensions](#data-quality-dimensions)
- [Testing Data with Great Expectations](#testing-data-with-great-expectations)
- [Testing Data with dbt Tests](#testing-data-with-dbt-tests)
- [Soda Core for Data Quality](#soda-core-for-data-quality)
- [Monitoring Pipelines with Monte Carlo](#monitoring-pipelines-with-monte-carlo)
- [Building Custom Alerting](#building-custom-alerting)
- [Data Quality in Practice](#data-quality-in-practice)
- [Incident Response Playbook](#incident-response-playbook)

---

## What is Data Observability?

**Data observability** is the ability to fully understand the health of the data in your system. It extends traditional software observability (metrics, logs, traces) to the data layer.

If software observability answers "is my application running?", data observability answers "is my data correct, fresh, and complete?"

### Why It Matters

The **cost of bad data** is enormous:
- ML models trained on dirty data make wrong predictions
- Business dashboards show wrong KPIs → wrong decisions
- Data engineers spend 40–80% of their time on data quality issues
- Trust in data erodes across the organization

### Data Downtime

Just like software has downtime (service unavailable), data has **data downtime** — periods when data is wrong, missing, or stale. Minimizing data downtime is the goal of data observability.

```
Software Downtime: "The website is down"
    → Measured in minutes, immediately visible

Data Downtime: "Revenue numbers are wrong for the last 3 days"
    → Measured in days, discovered late, high business impact
```

---

## The Five Pillars of Data Observability

Coined by Monte Carlo, these five pillars form the foundation:

### 1. Freshness
**Is data up to date?**
- When was the table last updated?
- Is there a gap between expected and actual update time?

```sql
-- Freshness check: has this table been updated in the last 2 hours?
SELECT
    table_name,
    MAX(updated_at) AS last_updated,
    DATEDIFF(minute, MAX(updated_at), CURRENT_TIMESTAMP) AS minutes_stale
FROM information_schema.tables
WHERE table_schema = 'analytics'
HAVING minutes_stale > 120  -- alert if stale > 2 hours
```

### 2. Distribution
**Does data look statistically normal?**
- Are there unexpected nulls?
- Are numeric values within expected ranges?
- Did a category suddenly appear or disappear?

```python
def check_distribution(df, column, expected_min, expected_max):
    actual_min = df[column].min()
    actual_max = df[column].max()
    null_pct = df[column].isna().mean() * 100

    issues = []
    if actual_min < expected_min:
        issues.append(f"Min {actual_min} below expected {expected_min}")
    if actual_max > expected_max:
        issues.append(f"Max {actual_max} above expected {expected_max}")
    if null_pct > 5:
        issues.append(f"Null rate {null_pct:.1f}% exceeds 5% threshold")
    return issues
```

### 3. Volume
**Is there the right amount of data?**
- Sudden drop → upstream failure, data loss
- Sudden spike → data duplication, bug in ingestion

```sql
-- Volume anomaly detection
WITH daily_counts AS (
    SELECT
        DATE(event_timestamp) AS event_date,
        COUNT(*) AS row_count
    FROM fct_events
    WHERE event_date >= CURRENT_DATE - 30
    GROUP BY 1
),
stats AS (
    SELECT
        AVG(row_count) AS avg_count,
        STDDEV(row_count) AS std_count
    FROM daily_counts
    WHERE event_date < CURRENT_DATE  -- exclude today
)
SELECT
    d.event_date,
    d.row_count,
    s.avg_count,
    ABS(d.row_count - s.avg_count) / NULLIF(s.std_count, 0) AS z_score
FROM daily_counts d, stats s
WHERE d.event_date = CURRENT_DATE
  AND ABS(d.row_count - s.avg_count) / NULLIF(s.std_count, 0) > 3
-- Alert if today's count is 3+ standard deviations from average
```

### 4. Schema
**Has the structure of the data changed unexpectedly?**
- Column dropped → downstream models break
- Data type changed → casting errors
- New column added → may indicate upstream changes

```python
def detect_schema_changes(old_schema: dict, new_schema: dict):
    added = set(new_schema.keys()) - set(old_schema.keys())
    removed = set(old_schema.keys()) - set(new_schema.keys())
    type_changed = {
        col for col in old_schema
        if col in new_schema and old_schema[col] != new_schema[col]
    }
    return {"added": added, "removed": removed, "type_changed": type_changed}
```

### 5. Lineage
**Where does data come from, and what depends on it?**
- When a table breaks, which downstream tables and dashboards are affected?
- When investigating bad data, trace it back to the root cause

```
Source: Postgres (orders table)
  → Airflow ETL job
    → S3 (raw/orders/)
      → Spark Transform
        → Snowflake (stg_orders)
          → dbt model (fct_orders)
            → dbt model (fct_revenue_daily)
              → Tableau Dashboard (Revenue Report)
              → ML Training Data
```

---

## Data Quality Dimensions

Beyond the 5 pillars, evaluate data quality across these 6 dimensions:

| Dimension | Question | Example Check |
|-----------|----------|---------------|
| **Completeness** | Is all expected data present? | No null user_ids |
| **Accuracy** | Is data correct? | Revenue > 0 |
| **Consistency** | Is data consistent across systems? | Orders in DB match Stripe |
| **Timeliness** | Is data fresh enough? | Updated within last hour |
| **Uniqueness** | Are there duplicates? | No duplicate order_ids |
| **Validity** | Does data match expected format? | Emails contain @ symbol |

---

## Testing Data with Great Expectations

**Great Expectations** (GX) is the most popular open-source data quality framework. It lets you define "expectations" (assertions) about your data and run them in your pipelines.

### Installation & Setup

```bash
pip install great_expectations
great_expectations init
```

### Creating Expectations

```python
import great_expectations as gx

context = gx.get_context()

# Connect to your data
datasource = context.sources.add_pandas_filesystem(
    name="local_data",
    base_directory="./data/"
)

# Create expectations suite
suite = context.add_expectation_suite("orders_suite")

# Define expectations
validator = context.get_validator(
    batch_request=...,
    expectation_suite_name="orders_suite"
)

# Completeness
validator.expect_column_values_to_not_be_null("order_id")
validator.expect_column_values_to_not_be_null("user_id")
validator.expect_column_values_to_not_be_null("order_date")

# Uniqueness
validator.expect_column_values_to_be_unique("order_id")

# Validity
validator.expect_column_values_to_be_between("order_amount", min_value=0, max_value=100000)
validator.expect_column_values_to_be_in_set("status", ["pending", "completed", "cancelled", "refunded"])
validator.expect_column_values_to_match_regex("email", r"[^@]+@[^@]+\.[^@]+")

# Volume
validator.expect_table_row_count_to_be_between(min_value=1000, max_value=10_000_000)

# Save the suite
validator.save_expectation_suite()
```

### Running Validations in Airflow

```python
from airflow.operators.python import PythonOperator
import great_expectations as gx

def run_gx_validation(**kwargs):
    context = gx.get_context()
    result = context.run_checkpoint(checkpoint_name="orders_checkpoint")

    if not result["success"]:
        failed = [r for r in result["run_results"].values() if not r["validation_result"]["success"]]
        raise ValueError(f"Data quality check FAILED: {len(failed)} expectations failed")

validate_task = PythonOperator(
    task_id="validate_orders",
    python_callable=run_gx_validation,
    dag=dag
)
```

---

## Testing Data with dbt Tests

dbt has a built-in testing framework. Tests run after model transformations and can block deployment if they fail.

### Generic Tests (Built-in)

```yaml
# models/schema.yml
version: 2

models:
  - name: fct_orders
    columns:
      - name: order_id
        tests:
          - unique
          - not_null
      - name: status
        tests:
          - accepted_values:
              values: ['pending', 'completed', 'cancelled', 'refunded']
      - name: user_id
        tests:
          - not_null
          - relationships:
              to: ref('dim_users')
              field: user_id
      - name: order_amount
        tests:
          - not_null
          - dbt_utils.expression_is_true:
              expression: ">= 0"
```

### Custom Singular Tests

```sql
-- tests/orders_no_future_dates.sql
-- This test fails if any orders have future dates
SELECT order_id
FROM {{ ref('fct_orders') }}
WHERE order_date > CURRENT_DATE

-- tests/revenue_reconciliation.sql
-- Revenue in our warehouse should match Stripe within 1%
SELECT
    ABS(warehouse_revenue - stripe_revenue) / NULLIF(stripe_revenue, 0) AS discrepancy_pct
FROM {{ ref('finance_reconciliation') }}
WHERE discrepancy_pct > 0.01
```

### Running Tests

```bash
# Run all tests
dbt test

# Run tests for a specific model
dbt test --select fct_orders

# Run tests with specific severity
dbt test --select "tag:critical"
```

---

## Soda Core for Data Quality

**Soda** provides SQL-based data quality checks with a simple YAML syntax.

```yaml
# checks.yml
checks for orders:
  - row_count > 0
  - missing_count(order_id) = 0
  - duplicate_count(order_id) = 0
  - invalid_count(status) = 0:
      valid values: [pending, completed, cancelled, refunded]
  - min(order_amount) >= 0
  - freshness(updated_at) < 2h
  - schema:
      name: Orders Schema
      fail:
        when required column missing: [order_id, user_id, order_date, status]
        when wrong column type:
          order_id: integer
          order_amount: decimal
```

```bash
# Run Soda checks
soda scan -d my_warehouse -c soda_config.yml checks.yml
```

---

## Monitoring Pipelines with Monte Carlo

**Monte Carlo** is the leading commercial data observability platform. It provides:
- **Automated anomaly detection** without writing manual rules
- **End-to-end lineage** from source to dashboard
- **Incident management** with root cause analysis

### What Monte Carlo Does Automatically

```
Learns your data patterns (volume, distributions, schema) over time
    ↓
Detects anomalies using ML models
    ↓
Creates incidents with affected downstream assets
    ↓
Notifies via Slack/PagerDuty/email
    ↓
Provides lineage to identify root cause
    ↓
Tracks incident resolution
```

### When to Use Monte Carlo vs Open-Source

| Scenario | Recommendation |
|----------|---------------|
| Small team, budget-conscious | Great Expectations + dbt tests |
| Many tables, complex lineage | Monte Carlo or Acceldata |
| Databricks-native | Databricks Lakehouse Monitoring |
| Snowflake-native | Snowflake Data Quality |
| Enterprise, compliance-heavy | Monte Carlo / Atlan |

---

## Building Custom Alerting

You don't need a commercial tool to get basic alerting. Build it yourself with SQL + Python.

### Freshness Alert

```python
import psycopg2
import requests

def check_table_freshness(table, expected_lag_hours=2):
    conn = psycopg2.connect("postgresql://...")
    cursor = conn.cursor()

    cursor.execute(f"""
        SELECT EXTRACT(EPOCH FROM (NOW() - MAX(updated_at))) / 3600 AS hours_stale
        FROM {table}
    """)
    hours_stale = cursor.fetchone()[0]

    if hours_stale > expected_lag_hours:
        send_slack_alert(
            f":red_circle: *Data Freshness Alert*\n"
            f"Table `{table}` is {hours_stale:.1f}h stale (threshold: {expected_lag_hours}h)"
        )

def send_slack_alert(message):
    requests.post(
        SLACK_WEBHOOK_URL,
        json={"text": message}
    )
```

### Volume Anomaly Alert

```python
def check_volume_anomaly(table, date_column, z_score_threshold=3.0):
    query = f"""
    WITH history AS (
        SELECT DATE({date_column}) AS d, COUNT(*) AS cnt
        FROM {table}
        WHERE {date_column} >= CURRENT_DATE - 30
        GROUP BY 1
    ),
    stats AS (SELECT AVG(cnt) avg, STDDEV(cnt) std FROM history WHERE d < CURRENT_DATE)
    SELECT h.cnt, s.avg, (h.cnt - s.avg) / NULLIF(s.std, 0) AS z
    FROM history h, stats s WHERE h.d = CURRENT_DATE
    """
    # Run query and check if z > threshold
    ...
```

---

## Data Quality in Practice

### The Data Quality Scorecard

Track quality across all critical tables in a central dashboard:

| Table | Freshness | Completeness | Uniqueness | Volume | Score |
|-------|-----------|-------------|------------|--------|-------|
| fct_orders | ✅ | ✅ | ✅ | ✅ | 100% |
| fct_events | ✅ | ⚠️ 2% null | ✅ | ✅ | 85% |
| dim_users | ❌ 5h stale | ✅ | ✅ | ✅ | 75% |
| fct_revenue | ✅ | ✅ | ✅ | ⚠️ -40% | 75% |

### Tiered Data Quality

Not all tables need the same SLA. Tier your tables:

```
Tier 1 (Critical) — Tier 1 SLA: <15 min data downtime, 99.9% quality
  → Revenue tables, customer-facing data, ML training data
  → Real-time monitoring, PagerDuty alerts

Tier 2 (Important) — SLA: <2h data downtime, 99% quality
  → Internal analytics, operational reports
  → Hourly checks, Slack alerts

Tier 3 (Exploratory) — Best effort
  → Ad-hoc analyses, experimental data
  → Daily checks, email alerts
```

---

## Incident Response Playbook

When a data quality incident happens, follow this playbook:

### Step 1: Detect & Triage (0–15 minutes)
```
1. Check alert → identify affected table(s)
2. Assess blast radius → what downstream tables, dashboards, models are affected?
3. Estimate business impact → who is affected? what decisions are being made on bad data?
4. Assign severity:
   - P0: Revenue data wrong, customer-facing broken
   - P1: Executive dashboards broken
   - P2: Analyst reports wrong
   - P3: Low-use table has issues
```

### Step 2: Communicate (15 minutes)
```
Post in #data-incidents:
  "🚨 [P1] Data Incident: fct_orders showing 0 revenue since 14:00 UTC
  Impact: Finance dashboard, Revenue ML model
  Investigating: @data-engineer-oncall
  ETA update: 30 min"
```

### Step 3: Investigate (15–60 minutes)
```
1. Check pipeline logs (Airflow, Spark)
2. Check upstream source data
3. Check recent code changes (git log)
4. Check schema changes
5. Isolate the failing step
```

### Step 4: Fix & Validate
```
1. Fix root cause
2. Backfill affected data if needed
3. Run data quality checks to confirm fix
4. Monitor for 1 hour
```

### Step 5: Post-Mortem
```
Document within 48 hours:
- Timeline of events
- Root cause
- Impact
- Fix applied
- Prevention: what tests/monitors would have caught this earlier?
```

---

*Previous: [17 — MLOps](17-mlops-ml-engineering.md) | Next: [19 — Data Governance](19-data-governance.md) →*
