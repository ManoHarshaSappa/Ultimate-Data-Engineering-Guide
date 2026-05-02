# 19 — Data Governance, Catalog & Lineage

> **Guide by [Mano Harsha Sappa](https://www.linkedin.com/in/manoharshasappa/)** | [Portfolio](https://manoharshasappa.github.io/portfolio_ManoHarshaSappa/) | [Back to Index](../README.md)

---

## Contents

- [What is Data Governance?](#what-is-data-governance)
- [Data Catalog](#data-catalog)
- [Data Lineage](#data-lineage)
- [Data Classification & Sensitivity](#data-classification--sensitivity)
- [Access Control & Security](#access-control--security)
- [Data Contracts](#data-contracts)
- [Privacy & Compliance (GDPR, CCPA)](#privacy--compliance-gdpr-ccpa)
- [Popular Data Governance Tools](#popular-data-governance-tools)
- [Building a Governance Program](#building-a-governance-program)

---

## What is Data Governance?

**Data governance** is the set of policies, processes, standards, and roles that ensure data is managed as a valuable, trustworthy, and secure asset throughout its lifecycle.

Think of data governance as the **rulebook for your data**:
- Who can access what data?
- How should sensitive data be handled?
- What does this column actually mean?
- Where did this number come from?
- How long should we keep this data?

### Why Data Governance Matters

Without governance:
- Nobody knows what a table contains without reading the code
- 5 teams have 5 different definitions of "active user"
- GDPR violations → massive fines
- Data breaches → reputational damage
- Data scientists waste 70% of their time finding and understanding data

### The Three Pillars of Data Governance

```
         DATA GOVERNANCE
        /        |        \
  Discoverability  Security  Accountability
     "I can find    "Only the    "Someone owns
      and understand  right people   this data and
      the data"       can see it"    is responsible"
```

---

## Data Catalog

A **data catalog** is the central metadata repository for your organization's data. It answers:
- What data do we have?
- What does each table and column mean?
- Who owns it?
- How fresh is it?
- Is it safe to use for my use case?

### What a Good Data Catalog Contains

```
Table: fct_orders
───────────────────────────────────────────
Description: One row per order placed on the platform.
             Includes all completed and cancelled orders.
Owner: Data Platform Team
Domain: Commerce
Tags: [core, revenue, PII]
SLA: Updated every 15 minutes
Row Count: 145,234,891
Last Updated: 2024-11-14 14:32 UTC

Columns:
  order_id       BIGINT    Unique identifier for each order. Never null.
  user_id        BIGINT    FK to dim_users. The buyer.
  order_date     DATE      Date the order was placed (UTC).
  order_amount   DECIMAL   Total order value in USD.
  status         VARCHAR   One of: pending, completed, cancelled, refunded
  created_at     TIMESTAMP When the record was created in the warehouse
  updated_at     TIMESTAMP When the record was last updated

Upstream sources: postgres.orders, stripe.charges
Downstream tables: fct_revenue_daily, agg_user_ltv
Dashboards: Revenue Dashboard, Finance Monthly Report
```

### Popular Data Catalogs

| Tool | Type | Best For |
|------|------|----------|
| **DataHub** | Open-source | Large orgs, rich integrations |
| **Apache Atlas** | Open-source | Hadoop/Hive ecosystems |
| **OpenMetadata** | Open-source | Modern data stacks |
| **Atlan** | Commercial | Modern data teams |
| **Alation** | Commercial | Enterprise, SQL-focused |
| **Collibra** | Commercial | Regulated industries |
| **dbt Docs** | Built into dbt | dbt-heavy stacks |

### Setting Up DataHub

```bash
# Install DataHub via Docker
pip install acryl-datahub
datahub docker quickstart

# Ingest metadata from dbt
cat > dbt_source.yml << EOF
source:
  type: dbt
  config:
    manifest_path: /path/to/target/manifest.json
    catalog_path: /path/to/target/catalog.json
    sources_path: /path/to/target/sources.json
    target_platform: snowflake
sink:
  type: datahub-rest
  config:
    server: "http://localhost:8080"
EOF

datahub ingest -c dbt_source.yml
```

### Documenting in dbt

dbt generates a catalog automatically from `schema.yml` descriptions:

```yaml
# models/schema.yml
version: 2

models:
  - name: fct_orders
    description: |
      One row per order placed on the platform.
      Includes completed, cancelled, and pending orders.
      Revenue recognized orders are in `fct_revenue`.
    meta:
      owner: "@data-platform"
      domain: "commerce"
      sla: "15 minutes"
    columns:
      - name: order_id
        description: Unique identifier for each order. Surrogate key from the source system.
        tests:
          - unique
          - not_null
      - name: order_amount
        description: Total order value in USD at the time of order. Excludes refunds.
      - name: status
        description: >
          Current order status. One of:
          - `pending`: order placed, payment not yet confirmed
          - `completed`: payment confirmed, order fulfilled
          - `cancelled`: order cancelled before fulfillment
          - `refunded`: order refunded after fulfillment
```

---

## Data Lineage

**Data lineage** tracks the journey of data from source to consumption — every transformation, join, and aggregation along the way.

### Column-Level Lineage

The most granular form of lineage — tracking individual columns through transformations:

```
Source: postgres.orders.total_price
  → Spark ETL: renamed to order_amount, cast to DECIMAL(10,2)
    → S3: raw/orders/order_amount
      → dbt stg_orders: passthrough
        → dbt fct_orders: order_amount (unchanged)
          → dbt fct_revenue_daily: SUM(order_amount) AS daily_revenue
            → Tableau: "Daily Revenue" metric
```

When "Daily Revenue" is wrong, you can trace it all the way back to `postgres.orders.total_price`.

### Why Lineage Matters

1. **Impact Analysis**: Before changing a source table, see what breaks downstream
2. **Root Cause Analysis**: Trace bad data to its origin
3. **Compliance**: Prove where PII came from (GDPR Article 30)
4. **Trust**: Analysts trust data more when they can see its origin

### Generating Lineage with dbt

dbt automatically generates lineage from `ref()` and `source()` calls:

```sql
-- dbt infers: fct_orders depends on stg_orders which depends on raw_orders source
SELECT
    o.order_id,
    o.order_amount,
    u.user_name,
    u.country
FROM {{ ref('stg_orders') }} o
JOIN {{ ref('dim_users') }} u ON o.user_id = u.user_id
```

Run `dbt docs generate && dbt docs serve` to see the lineage graph in your browser.

---

## Data Classification & Sensitivity

Classify all data by sensitivity level to enforce appropriate handling:

### Sensitivity Tiers

```
PUBLIC — Can be shared with anyone, inside or outside the company
  Examples: Product catalog, public pricing, company blog data

INTERNAL — For employees only, no external sharing
  Examples: Internal KPIs, aggregate metrics, anonymized analytics

CONFIDENTIAL — Restricted to teams with business need
  Examples: Revenue data, cost data, user behavior aggregates

RESTRICTED (PII) — Highest sensitivity, strict access controls
  Examples: Names, emails, phone numbers, addresses, SSNs, payment data
```

### Tagging PII in dbt

```yaml
models:
  - name: dim_users
    columns:
      - name: email
        meta:
          pii: true
          classification: "RESTRICTED"
      - name: full_name
        meta:
          pii: true
          classification: "RESTRICTED"
      - name: country
        meta:
          pii: false
          classification: "INTERNAL"
```

### Data Masking

Never expose raw PII where it's not needed:

```sql
-- Masking view for analysts who don't need real email addresses
CREATE VIEW dim_users_masked AS
SELECT
    user_id,
    -- Mask email: keep domain, hash local part
    CONCAT(MD5(email), '@', SPLIT_PART(email, '@', 2)) AS email_masked,
    -- Truncate name to first initial
    LEFT(full_name, 1) || '***' AS name_masked,
    country,
    subscription_tier,
    created_at
FROM dim_users;
```

---

## Access Control & Security

### Role-Based Access Control (RBAC)

Define roles and grant minimum necessary permissions:

```sql
-- Snowflake RBAC setup
-- Create roles
CREATE ROLE analyst_role;
CREATE ROLE data_engineer_role;
CREATE ROLE ml_engineer_role;
CREATE ROLE finance_role;

-- Grant table access per role
GRANT SELECT ON DATABASE analytics TO ROLE analyst_role;
GRANT SELECT ON SCHEMA analytics.public TO ROLE analyst_role;

-- Finance: only revenue tables, no PII
GRANT SELECT ON TABLE analytics.public.fct_revenue_daily TO ROLE finance_role;
REVOKE SELECT ON TABLE analytics.public.dim_users FROM ROLE finance_role;

-- ML Engineers: read feature tables, write ML schemas
GRANT SELECT ON DATABASE analytics TO ROLE ml_engineer_role;
GRANT ALL ON SCHEMA analytics.ml_features TO ROLE ml_engineer_role;

-- Assign roles to users
GRANT ROLE analyst_role TO USER john_doe;
GRANT ROLE ml_engineer_role TO USER alice_smith;
```

### Row-Level Security

Control which rows a user can see:

```sql
-- BigQuery Row-Level Security
CREATE ROW ACCESS POLICY country_filter
ON my_dataset.user_data
GRANT TO ("group:us-team@company.com")
FILTER USING (country = 'US');

CREATE ROW ACCESS POLICY eu_filter
ON my_dataset.user_data
GRANT TO ("group:eu-team@company.com")
FILTER USING (country IN ('DE', 'FR', 'UK'));
```

---

## Data Contracts

A **data contract** is a formal agreement between a data producer and data consumers about the structure, semantics, and SLAs of a dataset.

Think of it like an API contract, but for data.

### Why Data Contracts Matter

Without contracts:
- A backend team renames a column → 10 downstream pipelines break silently
- Schema changes are discovered by broken dashboards, not PRs
- No clear ownership or accountability

With contracts:
- Schema changes require a contract version bump
- Consumers are notified before breaking changes
- Producers are accountable to SLAs

### Data Contract Structure

```yaml
# data-contracts/orders.yml
id: com.company.orders.v2
version: 2.0.0
status: active

info:
  title: Orders Data Contract
  owner: data-platform@company.com
  domain: commerce
  description: Core orders table — one row per order

servers:
  production:
    type: snowflake
    database: ANALYTICS
    schema: PUBLIC
    table: FCT_ORDERS

terms:
  sla: "Updated every 15 minutes, 99.9% uptime"
  retention: "3 years"
  support: "Slack #data-platform"

schema:
  type: object
  required: [order_id, user_id, order_date, order_amount, status]
  properties:
    order_id:
      type: integer
      description: Unique order identifier
    user_id:
      type: integer
      description: FK to dim_users
    order_date:
      type: date
      description: Date the order was placed (UTC)
    order_amount:
      type: number
      minimum: 0
      description: Total order value in USD
    status:
      type: string
      enum: [pending, completed, cancelled, refunded]

quality:
  - type: completeness
    column: order_id
    threshold: 100%
  - type: uniqueness
    column: order_id
    threshold: 100%
  - type: freshness
    maxAge: 20 minutes
```

### Enforcing Data Contracts

```python
import yaml
import jsonschema

def validate_against_contract(df, contract_path):
    with open(contract_path) as f:
        contract = yaml.safe_load(f)

    schema = contract["schema"]
    required_columns = schema["required"]

    # Check required columns exist
    missing = set(required_columns) - set(df.columns)
    if missing:
        raise ValueError(f"Contract violation: missing columns {missing}")

    # Check row count freshness SLA
    max_age_minutes = 20
    most_recent = df["updated_at"].max()
    age_minutes = (pd.Timestamp.now() - most_recent).total_seconds() / 60
    if age_minutes > max_age_minutes:
        raise ValueError(f"Contract violation: data is {age_minutes:.0f}min old, max {max_age_minutes}min")

    print("Contract validation passed")
```

---

## Privacy & Compliance (GDPR, CCPA)

### Key Regulations

| Regulation | Scope | Key Requirements |
|-----------|-------|-----------------|
| **GDPR** | EU personal data | Right to erasure, data minimization, breach notification 72h |
| **CCPA** | California residents | Right to know, delete, opt-out of sale |
| **HIPAA** | US healthcare data | PHI encryption, audit logs, BAAs |
| **PCI DSS** | Payment card data | Tokenization, encryption, access controls |

### GDPR for Data Engineers

**Right to Erasure ("Right to be Forgotten")**:
A user can request all their data be deleted. You need a process to handle this.

```python
def delete_user_data(user_id: int):
    """Handle GDPR erasure request for a user."""

    # 1. Delete from production DB
    db.execute("DELETE FROM users WHERE user_id = %s", [user_id])
    db.execute("DELETE FROM orders WHERE user_id = %s", [user_id])

    # 2. Anonymize in data warehouse (can't delete from Snowflake partitioned tables easily)
    snowflake.execute("""
        UPDATE dim_users
        SET
            email = 'deleted_' || user_id || '@deleted.com',
            full_name = 'DELETED USER',
            phone = NULL,
            address = NULL,
            deleted_at = CURRENT_TIMESTAMP,
            is_deleted = TRUE
        WHERE user_id = %s
    """, [user_id])

    # 3. Remove from ML training datasets (important!)
    remove_from_ml_datasets(user_id)

    # 4. Log the erasure (required by GDPR)
    audit_log.insert({
        "action": "erasure_request",
        "user_id": user_id,
        "completed_at": datetime.now(),
        "requester": request.user_email
    })
```

**Data Minimization**: Only collect and retain data you actually need.

```sql
-- Anonymize user data older than 3 years (GDPR data minimization)
UPDATE dim_users
SET
    email = MD5(email),
    full_name = 'ANONYMIZED',
    phone = NULL
WHERE created_at < CURRENT_DATE - INTERVAL '3 years'
  AND is_active = FALSE;
```

---

## Popular Data Governance Tools

### Metadata & Catalog

| Tool | Strengths |
|------|----------|
| **DataHub (LinkedIn)** | Open-source, lineage, search, rich integrations |
| **OpenMetadata** | Modern UI, built-in quality, easy setup |
| **Apache Atlas** | Hadoop ecosystem, classification, lineage |
| **Atlan** | Modern SaaS, collaboration features |
| **Alation** | SQL queries, business glossary, mature product |
| **Collibra** | Enterprise, policy management, regulated industries |

### Access Control

| Tool | Use Case |
|------|----------|
| **Immuta** | Automated policy enforcement, multi-cloud |
| **Privacera** | ABAC, data marketplace |
| **AWS Lake Formation** | AWS-native, column/row security |
| **Snowflake RBAC** | Built-in, role hierarchy |

### Data Contracts

| Tool | Use Case |
|------|----------|
| **Schemata** | Open-source contract framework |
| **PayPal's Data Contract CLI** | CLI-based contract validation |
| **dbt contracts** | Schema-level contracts in dbt 1.5+ |

---

## Building a Governance Program

### Phase 1: Foundation (Month 1–3)

```
✅ Identify your most critical 20 tables (the "golden datasets")
✅ Document them: owner, description, column definitions
✅ Set up a basic data catalog (start with dbt docs if you use dbt)
✅ Classify data by sensitivity tier
✅ Implement RBAC in your data warehouse
```

### Phase 2: Quality & Lineage (Month 3–6)

```
✅ Add dbt tests to all golden datasets
✅ Set up freshness monitoring
✅ Generate lineage (via dbt or DataHub)
✅ Build a data quality scorecard
✅ Establish a data incident process
```

### Phase 3: Contracts & Compliance (Month 6–12)

```
✅ Write data contracts for Tier 1 datasets
✅ Implement PII tagging across all tables
✅ Build GDPR erasure process
✅ Set up column-level access controls
✅ Run a data privacy audit
```

### Phase 4: Mature Governance (12+ months)

```
✅ Data mesh: domain-owned governance
✅ Automated contract enforcement in CI/CD
✅ Data marketplace for internal data sharing
✅ Automated PII discovery
✅ Real-time policy enforcement
```

---

### The One-Person Governance Starter Kit

If you're a single data engineer starting from scratch:

1. **Week 1**: Document your top 10 tables in a shared Google Doc or Notion
2. **Week 2**: Set up dbt docs and add descriptions to all models and columns
3. **Week 3**: Classify all tables as PUBLIC / INTERNAL / CONFIDENTIAL / RESTRICTED
4. **Week 4**: Create roles in your warehouse and restrict PII access

This gets you 80% of the governance value with 20% of the effort.

---

*Previous: [18 — Data Observability](18-data-observability.md) | Next: [20 — Real-Time Analytics](20-real-time-analytics.md) →*
