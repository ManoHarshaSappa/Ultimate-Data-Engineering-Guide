# 17 — MLOps & ML Engineering for Data Engineers

> **Guide by [Mano Harsha Sappa](https://www.linkedin.com/in/manoharshasappa/)** | [Portfolio](https://manoharshasappa.github.io/portfolio_ManoHarshaSappa/) | [Back to Index](../README.md)

---

## Contents

- [What is MLOps?](#what-is-mlops)
- [The ML Lifecycle](#the-ml-lifecycle)
- [Data Engineer's Role in ML](#data-engineers-role-in-ml)
- [Feature Stores](#feature-stores)
- [Model Training Pipelines](#model-training-pipelines)
- [Model Serving & Inference](#model-serving--inference)
- [ML Monitoring & Drift Detection](#ml-monitoring--drift-detection)
- [Popular MLOps Tools](#popular-mlops-tools)
- [End-to-End MLOps Architecture](#end-to-end-mlops-architecture)
- [Practical Examples](#practical-examples)

---

## What is MLOps?

**MLOps** (Machine Learning Operations) is the set of practices, tools, and cultural philosophies that unify ML development and operations. It is to ML what DevOps is to software engineering.

Without MLOps, ML projects stall at the prototype stage. With MLOps:
- Models get deployed reliably and quickly
- Model performance is continuously monitored
- Retraining is automated when drift is detected
- Data lineage from raw input to prediction is tracked

### The MLOps Maturity Model

```
Level 0 — Manual ML
  → Jupyter notebooks, manual scripts, no automation
  → Models deployed by "copy-paste to server"
  → No monitoring, no retraining

Level 1 — ML Pipeline Automation
  → Automated training pipelines
  → Continuous training triggered by data or schedule
  → Experiment tracking with MLflow or W&B

Level 2 — CI/CD for ML
  → Automated model building, testing, deployment
  → A/B testing in production
  → Feature store in use
  → Full monitoring stack
```

Most companies sit at Level 0. Getting to Level 2 is the goal.

---

## The ML Lifecycle

```
Business Problem Definition
         ↓
Data Collection & Exploration
         ↓
Feature Engineering
         ↓
Model Training & Experimentation
         ↓
Model Evaluation & Validation
         ↓
Model Deployment (Serving)
         ↓
Monitoring & Observability
         ↓
Retraining (back to Feature Engineering)
```

Every arrow in this diagram is a potential engineering problem. Data engineers own the first three steps and support the rest.

---

## Data Engineer's Role in ML

Data engineers are the backbone of any successful ML project. Here's exactly what they own:

### 1. Training Data Pipelines
Build pipelines that produce clean, labeled, versioned datasets ready for model training.

```python
# Example: PySpark pipeline to prepare training data
from pyspark.sql import SparkSession
from pyspark.sql import functions as F

spark = SparkSession.builder.appName("TrainingDataPipeline").getOrCreate()

# Load raw events
events = spark.read.parquet("s3://data-lake/raw/user_events/")

# Build features
features = (
    events
    .filter(F.col("event_date") >= "2024-01-01")
    .groupBy("user_id")
    .agg(
        F.count("event_id").alias("total_events"),
        F.countDistinct("session_id").alias("total_sessions"),
        F.avg("session_duration_seconds").alias("avg_session_duration"),
        F.max("event_date").alias("last_active_date"),
        F.datediff(F.current_date(), F.max("event_date")).alias("days_since_active"),
    )
    .withColumn("is_churned", F.when(F.col("days_since_active") > 30, 1).otherwise(0))
)

# Save versioned dataset
features.write.mode("overwrite").parquet(
    "s3://data-lake/ml-datasets/churn-prediction/v3/"
)
```

### 2. Feature Engineering at Scale
Transform raw data into ML-ready features, often joining across many tables.

```sql
-- dbt model: fct_user_ml_features.sql
WITH user_events AS (
    SELECT
        user_id,
        COUNT(*) AS event_count_30d,
        COUNT(DISTINCT DATE(event_timestamp)) AS active_days_30d,
        SUM(CASE WHEN event_type = 'purchase' THEN revenue ELSE 0 END) AS revenue_30d,
        MAX(event_timestamp) AS last_event_at
    FROM {{ ref('fct_events') }}
    WHERE event_timestamp >= CURRENT_DATE - INTERVAL '30 days'
    GROUP BY user_id
),

user_profile AS (
    SELECT
        user_id,
        DATEDIFF(CURRENT_DATE, created_at) AS account_age_days,
        subscription_tier,
        country
    FROM {{ ref('dim_users') }}
)

SELECT
    e.user_id,
    e.event_count_30d,
    e.active_days_30d,
    e.revenue_30d,
    DATEDIFF(CURRENT_DATE, e.last_event_at) AS days_since_last_event,
    p.account_age_days,
    p.subscription_tier,
    p.country
FROM user_events e
JOIN user_profile p USING (user_id)
```

### 3. Data Versioning
Use tools like DVC or Delta Lake snapshots so models can be retrained on the exact same data.

```bash
# DVC example: track training datasets
dvc init
dvc add data/training/churn_features_v3.parquet
git add data/training/churn_features_v3.parquet.dvc .gitignore
git commit -m "Add churn training dataset v3"
dvc push  # pushes to S3/GCS
```

---

## Feature Stores

A **feature store** is a centralized repository for ML features. It solves the most common MLOps problems:
- Duplication (every team recomputes the same features)
- Training-serving skew (features computed differently in training vs production)
- Freshness (stale features reaching production models)

### Feature Store Architecture

```
Offline Store (Historical)          Online Store (Real-Time)
─────────────────────────          ─────────────────────────
S3 / BigQuery / Snowflake    ←→    Redis / DynamoDB / Cassandra
Daily/hourly batch jobs             Low-latency serving (<10ms)
Used for: model training            Used for: real-time inference
```

### Popular Feature Stores

| Tool | Type | Best For |
|------|------|----------|
| **Feast** | Open-source | General-purpose, cloud-agnostic |
| **Tecton** | Managed SaaS | Enterprise, real-time features |
| **Hopsworks** | Open-source | On-prem, full ML platform |
| **AWS SageMaker FS** | Managed | AWS-native stacks |
| **Databricks FS** | Managed | Databricks-native stacks |
| **Vertex AI FS** | Managed | GCP-native stacks |

### Feast Example

```python
from feast import FeatureStore

store = FeatureStore(repo_path=".")

# Retrieve features for model training
training_df = store.get_historical_features(
    entity_df=entity_df,
    features=[
        "user_stats:event_count_30d",
        "user_stats:revenue_30d",
        "user_stats:days_since_last_event",
        "user_profile:subscription_tier",
    ],
).to_df()

# Retrieve features for real-time inference
online_features = store.get_online_features(
    features=["user_stats:event_count_30d", "user_stats:revenue_30d"],
    entity_rows=[{"user_id": 12345}],
).to_dict()
```

---

## Model Training Pipelines

Model training pipelines automate the steps from data → trained model. They should be:
- **Reproducible**: same input always produces same output
- **Versioned**: track experiments, parameters, metrics
- **Automated**: triggered by schedule or data arrival

### MLflow for Experiment Tracking

```python
import mlflow
import mlflow.sklearn
from sklearn.ensemble import GradientBoostingClassifier
from sklearn.metrics import roc_auc_score
import pandas as pd

mlflow.set_experiment("churn-prediction")

with mlflow.start_run(run_name="gbm-v3"):
    # Log parameters
    params = {"n_estimators": 200, "max_depth": 5, "learning_rate": 0.05}
    mlflow.log_params(params)

    # Train model
    model = GradientBoostingClassifier(**params)
    model.fit(X_train, y_train)

    # Log metrics
    auc = roc_auc_score(y_test, model.predict_proba(X_test)[:, 1])
    mlflow.log_metric("roc_auc", auc)

    # Log model
    mlflow.sklearn.log_model(model, "churn_model")
    print(f"Run ID: {mlflow.active_run().info.run_id}")
    print(f"AUC: {auc:.4f}")
```

### Airflow ML Training DAG

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.providers.amazon.aws.operators.sagemaker import SageMakerTrainingOperator
from datetime import datetime

def prepare_training_data(**kwargs):
    # Run Spark job to prepare features
    pass

def validate_training_data(**kwargs):
    # Check row count, nulls, distributions
    pass

def register_model(**kwargs):
    # Push model to MLflow Model Registry
    pass

with DAG("ml_training_pipeline", schedule_interval="@weekly", start_date=datetime(2024, 1, 1)) as dag:

    prepare = PythonOperator(task_id="prepare_data", python_callable=prepare_training_data)

    validate = PythonOperator(task_id="validate_data", python_callable=validate_training_data)

    train = SageMakerTrainingOperator(
        task_id="train_model",
        config={...},  # SageMaker training config
    )

    register = PythonOperator(task_id="register_model", python_callable=register_model)

    prepare >> validate >> train >> register
```

---

## Model Serving & Inference

After training, models must be deployed to serve predictions. There are two patterns:

### Batch Inference
Run predictions on a schedule and store results in a database. Best for non-real-time use cases.

```python
# Airflow DAG: batch scoring
def score_users(**kwargs):
    import mlflow
    import pandas as pd

    # Load production model
    model = mlflow.sklearn.load_model("models:/churn-prediction/Production")

    # Load users to score
    users_df = pd.read_parquet("s3://data-lake/ml-datasets/users_to_score/")

    # Score
    users_df["churn_probability"] = model.predict_proba(
        users_df[FEATURE_COLUMNS]
    )[:, 1]

    # Write predictions
    users_df[["user_id", "churn_probability", "scored_at"]].to_parquet(
        "s3://data-lake/predictions/churn/daily/",
        partition_cols=["scored_at"]
    )
```

### Real-Time Inference
Serve predictions via an API with sub-100ms latency. Best for interactive apps.

```python
# FastAPI model serving endpoint
from fastapi import FastAPI
from pydantic import BaseModel
import mlflow
import numpy as np

app = FastAPI()
model = mlflow.sklearn.load_model("models:/churn-prediction/Production")

class UserFeatures(BaseModel):
    event_count_30d: int
    revenue_30d: float
    days_since_last_event: int
    subscription_tier: str

@app.post("/predict/churn")
def predict_churn(features: UserFeatures):
    X = np.array([[
        features.event_count_30d,
        features.revenue_30d,
        features.days_since_last_event,
        {"free": 0, "pro": 1, "enterprise": 2}[features.subscription_tier]
    ]])
    prob = model.predict_proba(X)[0][1]
    return {"churn_probability": round(float(prob), 4)}
```

---

## ML Monitoring & Drift Detection

Models degrade over time as the world changes. You must monitor:

### Types of Drift

| Type | Definition | Example |
|------|-----------|---------|
| **Data Drift** | Input feature distribution shifts | Users start using mobile more than desktop |
| **Label Drift** | Target variable distribution shifts | Churn rate drops from 10% to 2% |
| **Concept Drift** | Relationship between features and target changes | Old churn signals no longer predict churn |
| **Prediction Drift** | Model output distribution shifts | Model starts predicting more churn |

### Monitoring with Evidently AI

```python
from evidently.report import Report
from evidently.metric_preset import DataDriftPreset, ClassificationPreset
import pandas as pd

# Reference data (training set) vs current data (production)
reference = pd.read_parquet("s3://ml/reference_data/churn_features.parquet")
current = pd.read_parquet("s3://ml/current_data/churn_features_today.parquet")

# Generate drift report
report = Report(metrics=[DataDriftPreset(), ClassificationPreset()])
report.run(reference_data=reference, current_data=current)
report.save_html("drift_report.html")

# Check if drift threshold exceeded
drift_results = report.as_dict()
drift_detected = drift_results["metrics"][0]["result"]["dataset_drift"]

if drift_detected:
    # Trigger alert and potential retraining
    send_slack_alert("Data drift detected in churn model — consider retraining")
```

---

## Popular MLOps Tools

### Experiment Tracking
| Tool | Description |
|------|-------------|
| **MLflow** | Open-source, self-hosted, universal |
| **Weights & Biases** | Managed, excellent visualization |
| **Neptune.ai** | Managed, metadata store |
| **Comet ML** | Managed, collaboration-focused |

### Pipeline Orchestration
| Tool | Description |
|------|-------------|
| **Airflow** | Best for data-heavy ML pipelines |
| **Kubeflow Pipelines** | Kubernetes-native ML pipelines |
| **ZenML** | Framework-agnostic ML pipelines |
| **Prefect** | Modern, Python-native pipelines |
| **Metaflow** | Netflix's ML pipeline framework |

### Model Registry & Deployment
| Tool | Description |
|------|-------------|
| **MLflow Model Registry** | Version, stage, deploy models |
| **BentoML** | Package and serve models |
| **Seldon Core** | Kubernetes model serving |
| **Ray Serve** | Scalable model serving |
| **AWS SageMaker** | Fully managed end-to-end |

---

## End-to-End MLOps Architecture

```
Data Sources (DB, API, Files)
          ↓
[DATA PIPELINE — Airflow + Spark]
          ↓
Feature Store (Feast/Tecton)
   /            \
Offline Store   Online Store
(S3/BigQuery)   (Redis)
   ↓                ↓
[TRAINING]      [INFERENCE API]
MLflow tracks    FastAPI / BentoML
experiments      reads online features
   ↓                ↓
Model Registry  Real-Time Predictions
(MLflow/Sagemaker)
   ↓
[MONITORING — Evidently/Grafana]
   ↓ (drift detected)
Auto-Retraining Trigger (Airflow DAG)
```

---

## Practical Examples

### Full Churn Prediction Pipeline

**Step 1: Feature engineering (dbt)**
```sql
-- models/ml/user_churn_features.sql
SELECT
    user_id,
    COUNT(*) OVER (PARTITION BY user_id ORDER BY event_date ROWS BETWEEN 29 PRECEDING AND CURRENT ROW) AS events_30d,
    SUM(revenue) OVER (PARTITION BY user_id ORDER BY event_date ROWS BETWEEN 29 PRECEDING AND CURRENT ROW) AS revenue_30d,
    DATEDIFF(CURRENT_DATE, MAX(event_date)) AS days_since_last,
    subscription_tier
FROM {{ ref('fct_events') }}
JOIN {{ ref('dim_users') }} USING (user_id)
```

**Step 2: Train and track (Python + MLflow)**
```python
with mlflow.start_run():
    model = XGBClassifier(n_estimators=300, max_depth=6)
    model.fit(X_train, y_train)
    mlflow.log_metric("auc", roc_auc_score(y_test, model.predict_proba(X_test)[:,1]))
    mlflow.xgboost.log_model(model, "model")
```

**Step 3: Register and deploy**
```python
client = mlflow.tracking.MlflowClient()
client.transition_model_version_stage(
    name="churn-prediction",
    version=3,
    stage="Production"
)
```

**Step 4: Batch scoring (Airflow)**
Score all users nightly, write to `predictions.churn_scores` table.

**Step 5: Monitor (daily drift check)**
Compare today's feature distributions to training set. Alert if drift > 10%.

---

### Key Takeaways for Data Engineers

1. **Own your features** — build reliable, versioned feature pipelines
2. **Track everything** — use MLflow or W&B from day one
3. **Automate retraining** — don't rely on humans to notice drift
4. **Test ML pipelines like software** — unit tests, integration tests, data quality checks
5. **Treat models as artifacts** — version them, test them before promoting to production

---

*Previous: [16 — Projects](16-projects.md) | Next: [18 — Data Observability](18-data-observability.md) →*
