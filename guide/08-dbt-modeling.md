# 08 — dbt Data Modeling Guide

> **Guide by [Mano Harsha Sappa](https://www.linkedin.com/in/manoharshasappa/)** | [Portfolio](https://manoharshasappa.github.io/portfolio_ManoHarshaSappa/) | [Back to Index](../README.md)

---

## Contents

- [Kimball Dimensional Modeling](#kimball-dimensional-modeling)
- [Step 1: Source Data](#step-1-source-data)
- [Step 2: Staging Layer](#step-2-staging-layer)
- [Step 3: Snapshots (SCD Type 2)](#step-3-snapshots-scd-type-2)
- [Step 4: Dimension Tables](#step-4-dimension-tables)
- [Step 5: Fact Tables](#step-5-fact-tables)
- [Step 6: Mart Layer](#step-6-mart-layer)
- [dbt Testing Strategy](#dbt-testing-strategy)
- [dbt Documentation](#dbt-documentation)
- [dbt Best Practices](#dbt-best-practices)

---

## Kimball Dimensional Modeling

The Kimball methodology (from Ralph Kimball's "Data Warehouse Toolkit") is the industry standard for organizing data in a warehouse for analytics.

### Core Concepts

**Fact Tables**: Record business events and measurements.
- Contain measurable, numeric data (revenue, quantity, duration)
- Usually append-only and very large
- Connected to dimension tables via foreign keys

**Dimension Tables**: Describe the context of facts.
- WHO (customer, employee)
- WHAT (product, service)
- WHERE (location, store)
- WHEN (date, time — the date dimension)
- HOW (channel, payment method)

**Star Schema**: A fact table surrounded by dimension tables.

```
              DIM_DATE ──────────────────────────────────────────┐
                                                                  │
DIM_CUSTOMER ──────────────────┐                                 │
                                │                                 │
DIM_PRODUCT ────────────────── FCT_ORDERS ──────────────────────┘
                                │                                 │
DIM_STORE ─────────────────────┘                                 │
                                                                  │
DIM_CHANNEL ──────────────────────────────────────────────────────┘
```

**Slowly Changing Dimensions (SCD)**:
- **Type 1**: Overwrite old value (lose history) — simple but no history
- **Type 2**: Add new row with new value, mark old row as expired (full history) — most common
- **Type 3**: Add new column for new value (limited history)

---

## The E-Commerce Case Study

We'll model data for an e-commerce store with a mobile app. Source tables in production database:

```
Production DB:
├── users           (user accounts)
├── products        (product catalog, prices change over time)
├── orders          (order headers)
├── order_items     (line items within orders)
└── payments        (payment transactions)
```

Business goals:
- Track daily revenue and margin
- Analyze customer behavior over time
- Understand product performance
- Track customer lifetime value

---

## Step 1: Source Data

Define your raw source tables in dbt so transformations reference them properly.

```yaml
# models/staging/_sources.yml
version: 2

sources:
  - name: ecommerce_db
    description: "Production e-commerce database"
    database: raw
    schema: public

    freshness:
      warn_after: {count: 24, period: hour}
      error_after: {count: 48, period: hour}
    loaded_at_field: updated_at

    tables:
      - name: users
        description: "User accounts"
        columns:
          - name: id
            tests: [not_null, unique]
          - name: email
            tests: [not_null, unique]

      - name: products
        description: "Product catalog"
        columns:
          - name: id
            tests: [not_null, unique]
          - name: price
            tests:
              - not_null
              - dbt_utils.expression_is_true:
                  expression: "> 0"

      - name: orders
        description: "Order headers"
        freshness:
          warn_after: {count: 6, period: hour}
        columns:
          - name: id
            tests: [not_null, unique]
          - name: user_id
            tests:
              - not_null
              - relationships:
                  to: source('ecommerce_db', 'users')
                  field: id

      - name: order_items
        columns:
          - name: id
            tests: [not_null, unique]
          - name: order_id
            tests:
              - not_null
              - relationships:
                  to: source('ecommerce_db', 'orders')
                  field: id

      - name: payments
        columns:
          - name: id
            tests: [not_null, unique]
```

---

## Step 2: Staging Layer

The staging layer is your Silver layer — cleaned, renamed, type-cast raw data. One staging model per source table. No business logic here.

**Rules for staging models**:
1. Name format: `stg_{source}__{table}.sql` (double underscore separates source from table)
2. Rename columns to consistent naming conventions
3. Cast types correctly (strings → timestamps, cents → dollars, etc.)
4. Filter out obviously invalid records
5. No joins across sources (that's for intermediate/mart layer)
6. Should be views (cheap, always fresh)

```sql
-- models/staging/stg_ecommerce__users.sql
WITH source AS (
    SELECT * FROM {{ source('ecommerce_db', 'users') }}
),

staged AS (
    SELECT
        -- Primary key
        id AS user_id,

        -- Dimensions
        first_name,
        last_name,
        CONCAT(first_name, ' ', last_name) AS full_name,
        LOWER(TRIM(email)) AS email,
        phone,
        country_code,

        -- Dates
        CAST(created_at AS TIMESTAMP) AS created_at,
        CAST(updated_at AS TIMESTAMP) AS updated_at,
        CAST(deleted_at AS TIMESTAMP) AS deleted_at,

        -- Flags
        is_active,
        email_verified,
        CASE WHEN deleted_at IS NOT NULL THEN TRUE ELSE FALSE END AS is_deleted

    FROM source
    WHERE id IS NOT NULL
)

SELECT * FROM staged
```

```sql
-- models/staging/stg_ecommerce__orders.sql
WITH source AS (
    SELECT * FROM {{ source('ecommerce_db', 'orders') }}
),

staged AS (
    SELECT
        id AS order_id,
        user_id,
        status AS order_status,
        LOWER(status) AS order_status_normalized,
        shipping_address_id,
        promo_code,
        CAST(ordered_at AS TIMESTAMP) AS ordered_at,
        CAST(shipped_at AS TIMESTAMP) AS shipped_at,
        CAST(delivered_at AS TIMESTAMP) AS delivered_at,
        CAST(returned_at AS TIMESTAMP) AS returned_at,
        CAST(created_at AS TIMESTAMP) AS created_at,

        -- Derived fields
        DATE(ordered_at) AS order_date,
        CASE
            WHEN returned_at IS NOT NULL THEN 'returned'
            WHEN delivered_at IS NOT NULL THEN 'delivered'
            WHEN shipped_at IS NOT NULL THEN 'shipped'
            ELSE order_status
        END AS fulfillment_status

    FROM source
    WHERE id IS NOT NULL
        AND status IN ('pending', 'processing', 'shipped', 'delivered', 'returned', 'cancelled')
)

SELECT * FROM staged
```

```sql
-- models/staging/stg_ecommerce__order_items.sql
WITH source AS (
    SELECT * FROM {{ source('ecommerce_db', 'order_items') }}
),

staged AS (
    SELECT
        id AS order_item_id,
        order_id,
        product_id,
        quantity,
        unit_price / 100.0 AS unit_price,        -- Convert cents to dollars
        unit_cost / 100.0 AS unit_cost,
        quantity * (unit_price / 100.0) AS item_revenue,
        quantity * (unit_cost / 100.0) AS item_cost,
        quantity * (unit_price / 100.0) - quantity * (unit_cost / 100.0) AS item_margin,
        discount_amount / 100.0 AS discount_amount,
        CAST(created_at AS TIMESTAMP) AS created_at
    FROM source
    WHERE id IS NOT NULL
        AND quantity > 0
        AND unit_price > 0
)

SELECT * FROM staged
```

---

## Step 3: Snapshots (SCD Type 2)

Snapshots track slowly changing dimensions — when a row in the source changes, dbt creates a new version with `dbt_valid_from` and `dbt_valid_to` timestamps.

```sql
-- snapshots/products_snapshot.sql
{% snapshot products_snapshot %}

{{
    config(
      target_schema='snapshots',
      unique_key='id',
      strategy='timestamp',      -- Use 'timestamp' or 'check'
      updated_at='updated_at',
    )
}}

SELECT
    id,
    name,
    price,
    cost,
    category,
    is_active,
    updated_at
FROM {{ source('ecommerce_db', 'products') }}

{% endsnapshot %}
```

This creates a table with:
- `dbt_scd_id`: Unique ID per version
- `dbt_valid_from`: When this version became active
- `dbt_valid_to`: When this version expired (NULL = current)
- `dbt_updated_at`: When the snapshot was last updated

```sql
-- Query: What was the product price at time of order?
SELECT
    o.order_id,
    p.name AS product_name,
    oi.unit_price AS charged_price,
    ps.price AS listed_price_at_order_time
FROM stg_orders o
JOIN stg_order_items oi ON o.order_id = oi.order_id
JOIN products_snapshot ps ON
    oi.product_id = ps.id
    AND o.ordered_at BETWEEN ps.dbt_valid_from
        AND COALESCE(ps.dbt_valid_to, CURRENT_TIMESTAMP)
```

---

## Step 4: Dimension Tables

Dimension tables provide descriptive context for fact table records.

```sql
-- models/marts/dim_customers.sql
{{
    config(
        materialized='table',
        unique_key='customer_key'
    )
}}

WITH customers AS (
    SELECT * FROM {{ ref('stg_ecommerce__users') }}
    WHERE NOT is_deleted
),

orders AS (
    SELECT * FROM {{ ref('stg_ecommerce__orders') }}
    WHERE order_status NOT IN ('cancelled')
),

order_metrics AS (
    SELECT
        user_id,
        MIN(order_date) AS first_order_date,
        MAX(order_date) AS most_recent_order_date,
        COUNT(*) AS lifetime_order_count
    FROM orders
    GROUP BY 1
),

final AS (
    SELECT
        -- Surrogate key (stable ID for this row in the warehouse)
        {{ dbt_utils.generate_surrogate_key(['c.user_id']) }} AS customer_key,

        -- Natural key (ID in source system)
        c.user_id,

        -- Descriptive attributes
        c.full_name,
        c.email,
        c.phone,
        c.country_code,
        c.created_at AS customer_since,

        -- Order history
        m.first_order_date,
        m.most_recent_order_date,
        COALESCE(m.lifetime_order_count, 0) AS lifetime_order_count,

        -- Segments
        CASE
            WHEN m.lifetime_order_count IS NULL THEN 'prospect'
            WHEN m.lifetime_order_count = 1 THEN 'new'
            WHEN m.lifetime_order_count >= 10 THEN 'vip'
            ELSE 'returning'
        END AS customer_segment,

        -- Is the customer active in last 90 days?
        CASE
            WHEN m.most_recent_order_date >= CURRENT_DATE - 90 THEN TRUE
            ELSE FALSE
        END AS is_active_90d,

        -- SCD tracking
        CURRENT_TIMESTAMP AS _dbt_loaded_at

    FROM customers c
    LEFT JOIN order_metrics m ON c.user_id = m.user_id
)

SELECT * FROM final
```

```sql
-- models/marts/dim_products.sql
-- Uses snapshot for historical product prices

{{
    config(
        materialized='table',
        unique_key='product_key'
    )
}}

WITH products AS (
    -- Current version of each product
    SELECT *
    FROM {{ ref('products_snapshot') }}
    WHERE dbt_valid_to IS NULL  -- Current records only
),

final AS (
    SELECT
        {{ dbt_utils.generate_surrogate_key(['id']) }} AS product_key,
        id AS product_id,
        name AS product_name,
        category,
        price AS current_price,
        cost AS current_cost,
        ROUND(((price - cost) / NULLIF(price, 0)) * 100, 2) AS gross_margin_pct,
        is_active,
        dbt_valid_from AS price_effective_from
    FROM products
)

SELECT * FROM final
```

```sql
-- models/marts/dim_date.sql
-- The date dimension — every analytics warehouse needs this

{{
    config(
        materialized='table'
    )
}}

WITH date_spine AS (
    {{ dbt_utils.date_spine(
        datepart="day",
        start_date="cast('2020-01-01' as date)",
        end_date="cast('2030-12-31' as date)"
    ) }}
),

final AS (
    SELECT
        date_day AS date_key,
        date_day,
        EXTRACT(YEAR FROM date_day) AS year,
        EXTRACT(QUARTER FROM date_day) AS quarter,
        EXTRACT(MONTH FROM date_day) AS month,
        EXTRACT(WEEK FROM date_day) AS week_of_year,
        EXTRACT(DAY FROM date_day) AS day_of_month,
        EXTRACT(DOW FROM date_day) AS day_of_week,  -- 0=Sunday
        TO_CHAR(date_day, 'Month') AS month_name,
        TO_CHAR(date_day, 'Day') AS day_name,
        CASE WHEN EXTRACT(DOW FROM date_day) IN (0, 6) THEN FALSE ELSE TRUE END AS is_weekday,
        CONCAT(EXTRACT(YEAR FROM date_day), '-Q', EXTRACT(QUARTER FROM date_day)) AS quarter_label,
        TO_CHAR(date_day, 'YYYY-MM') AS year_month
    FROM date_spine
)

SELECT * FROM final
```

---

## Step 5: Fact Tables

Fact tables record business events. They are the heart of your data model.

```sql
-- models/marts/fct_orders.sql
{{
    config(
        materialized='incremental',
        unique_key='order_key',
        on_schema_change='append_new_columns',
        partition_by={
            "field": "order_date",
            "data_type": "date",
            "granularity": "day"
        }
    )
}}

WITH orders AS (
    SELECT * FROM {{ ref('stg_ecommerce__orders') }}
    {% if is_incremental() %}
        WHERE ordered_at > (SELECT MAX(ordered_at) FROM {{ this }})
    {% endif %}
),

order_items AS (
    SELECT * FROM {{ ref('stg_ecommerce__order_items') }}
),

order_aggregates AS (
    SELECT
        order_id,
        COUNT(*) AS item_count,
        SUM(item_revenue) AS gross_revenue,
        SUM(item_cost) AS total_cost,
        SUM(item_margin) AS gross_margin,
        SUM(discount_amount) AS total_discount
    FROM order_items
    GROUP BY 1
),

customers AS (
    SELECT user_id, customer_key FROM {{ ref('dim_customers') }}
),

final AS (
    SELECT
        -- Surrogate key
        {{ dbt_utils.generate_surrogate_key(['o.order_id']) }} AS order_key,

        -- Natural keys
        o.order_id,
        o.user_id,

        -- Foreign keys to dimensions
        c.customer_key,
        o.order_date AS date_key,

        -- Order facts (measurements)
        agg.item_count,
        agg.gross_revenue,
        agg.total_cost,
        agg.gross_margin,
        ROUND(agg.gross_margin / NULLIF(agg.gross_revenue, 0) * 100, 2) AS gross_margin_pct,
        agg.total_discount,
        agg.gross_revenue - agg.total_discount AS net_revenue,

        -- Status flags
        o.order_status,
        o.fulfillment_status,
        CASE WHEN o.returned_at IS NOT NULL THEN TRUE ELSE FALSE END AS is_returned,

        -- Timestamps
        o.ordered_at,
        o.shipped_at,
        o.delivered_at,
        o.returned_at,
        o.order_date

    FROM orders o
    LEFT JOIN order_aggregates agg ON o.order_id = agg.order_id
    LEFT JOIN customers c ON o.user_id = c.user_id
)

SELECT * FROM final
```

```sql
-- models/marts/fct_order_items.sql
-- Grain: one row per order line item

{{
    config(
        materialized='incremental',
        unique_key='order_item_key'
    )
}}

WITH order_items AS (
    SELECT * FROM {{ ref('stg_ecommerce__order_items') }}
    {% if is_incremental() %}
        WHERE created_at > (SELECT MAX(created_at) FROM {{ this }})
    {% endif %}
),

products AS (
    SELECT product_id, product_key, product_name, category, gross_margin_pct
    FROM {{ ref('dim_products') }}
),

orders AS (
    SELECT order_id, order_key, customer_key, order_date
    FROM {{ ref('fct_orders') }}
),

final AS (
    SELECT
        {{ dbt_utils.generate_surrogate_key(['oi.order_item_id']) }} AS order_item_key,
        oi.order_item_id,
        o.order_key,
        o.customer_key,
        p.product_key,
        o.order_date AS date_key,
        oi.quantity,
        oi.unit_price,
        oi.unit_cost,
        oi.item_revenue,
        oi.item_cost,
        oi.item_margin,
        oi.discount_amount,
        oi.created_at
    FROM order_items oi
    LEFT JOIN orders o ON oi.order_id = o.order_id
    LEFT JOIN products p ON oi.product_id = p.product_id
)

SELECT * FROM final
```

---

## Step 6: Mart Layer

The mart layer has business-specific aggregations — Gold layer tables optimized for specific analytics use cases.

```sql
-- models/marts/finance/daily_revenue_summary.sql
{{
    config(
        materialized='incremental',
        unique_key='report_date'
    )
}}

WITH daily AS (
    SELECT
        order_date AS report_date,
        COUNT(*) AS order_count,
        COUNT(DISTINCT user_id) AS unique_customers,
        SUM(gross_revenue) AS gross_revenue,
        SUM(net_revenue) AS net_revenue,
        SUM(gross_margin) AS gross_margin,
        SUM(CASE WHEN is_returned THEN gross_revenue ELSE 0 END) AS returned_revenue,
        AVG(net_revenue) AS avg_order_value,
        SUM(gross_margin) / NULLIF(SUM(gross_revenue), 0) * 100 AS gross_margin_pct
    FROM {{ ref('fct_orders') }}
    WHERE order_status NOT IN ('cancelled')
    {% if is_incremental() %}
        AND order_date >= CURRENT_DATE - 7  -- Recalculate last 7 days for corrections
    {% endif %}
    GROUP BY 1
)

SELECT
    report_date,
    order_count,
    unique_customers,
    gross_revenue,
    net_revenue,
    gross_margin,
    gross_margin_pct,
    returned_revenue,
    avg_order_value,
    -- Running totals
    SUM(net_revenue) OVER (PARTITION BY DATE_TRUNC('month', report_date)
        ORDER BY report_date) AS mtd_revenue,
    SUM(net_revenue) OVER (PARTITION BY DATE_TRUNC('year', report_date)
        ORDER BY report_date) AS ytd_revenue
FROM daily
```

```sql
-- models/marts/marketing/customer_ltv.sql
-- Customer Lifetime Value analysis

{{
    config(materialized='table')
}}

WITH customer_orders AS (
    SELECT
        customer_key,
        user_id,
        COUNT(*) AS total_orders,
        SUM(net_revenue) AS total_revenue,
        MIN(ordered_at) AS first_order_at,
        MAX(ordered_at) AS last_order_at,
        DATEDIFF('day', MIN(ordered_at), MAX(ordered_at)) AS customer_lifespan_days
    FROM {{ ref('fct_orders') }}
    WHERE order_status NOT IN ('cancelled')
    GROUP BY 1, 2
),

customers AS (
    SELECT * FROM {{ ref('dim_customers') }}
),

final AS (
    SELECT
        c.user_id,
        c.email,
        c.customer_segment,
        co.total_orders,
        co.total_revenue AS lifetime_value,
        co.total_revenue / co.total_orders AS avg_order_value,
        co.customer_lifespan_days,
        CASE
            WHEN co.total_revenue > 1000 THEN 'High LTV'
            WHEN co.total_revenue > 250 THEN 'Medium LTV'
            ELSE 'Low LTV'
        END AS ltv_tier,
        co.first_order_at,
        co.last_order_at,
        CURRENT_DATE - co.last_order_at::DATE AS days_since_last_order
    FROM customers c
    LEFT JOIN customer_orders co ON c.customer_key = co.customer_key
)

SELECT * FROM final
```

---

## dbt Testing Strategy

Testing is what separates professional dbt projects from amateur ones.

### Built-in Generic Tests

```yaml
# models/marts/_schema.yml
version: 2

models:
  - name: fct_orders
    tests:
      - dbt_utils.equal_rowcount:
          compare_model: ref('stg_ecommerce__orders')

    columns:
      - name: order_key
        tests:
          - not_null
          - unique

      - name: gross_revenue
        tests:
          - not_null
          - dbt_utils.expression_is_true:
              expression: ">= 0"

      - name: order_status
        tests:
          - accepted_values:
              values: ['pending', 'processing', 'shipped', 'delivered', 'returned', 'cancelled']

      - name: customer_key
        tests:
          - not_null
          - relationships:
              to: ref('dim_customers')
              field: customer_key
```

### Custom Tests

```sql
-- tests/test_revenue_never_decreases_year_over_year.sql
-- Fail if current year revenue is less than 50% of prior year

WITH current_year AS (
    SELECT SUM(net_revenue) AS revenue
    FROM {{ ref('daily_revenue_summary') }}
    WHERE EXTRACT(YEAR FROM report_date) = EXTRACT(YEAR FROM CURRENT_DATE)
),
prior_year AS (
    SELECT SUM(net_revenue) AS revenue
    FROM {{ ref('daily_revenue_summary') }}
    WHERE EXTRACT(YEAR FROM report_date) = EXTRACT(YEAR FROM CURRENT_DATE) - 1
)
SELECT 1
FROM current_year, prior_year
WHERE current_year.revenue < prior_year.revenue * 0.5
```

### Test Coverage Targets

| Layer | Critical Tests | Nice-to-have |
|-------|--------------|--------------|
| Staging | not_null on PKs, unique on PKs | accepted_values, relationships |
| Dimensions | unique surrogate key, not_null FKs | all descriptive columns |
| Facts | unique surrogate key, not_null FKs, revenue >= 0 | aggregation correctness |
| Marts | rowcount matches, aggregations make sense | business rule tests |

---

## dbt Documentation

Good documentation makes your dbt project self-service for analysts and data scientists.

```yaml
# models/marts/_schema.yml
models:
  - name: fct_orders
    description: |
      **Grain**: One row per order.

      This table contains all completed and in-progress orders from the
      e-commerce platform. Cancelled orders are excluded.

      **Join Keys**:
      - `customer_key` → `dim_customers.customer_key`
      - `date_key` → `dim_date.date_key`

      **Updated**: Incrementally, hourly (orders placed in last 24 hours)

      **Owner**: Data Engineering Team (mano@company.com)
```

Generate and serve docs:
```bash
dbt docs generate   # Creates target/catalog.json + target/manifest.json
dbt docs serve      # Serves at http://localhost:8080
```

The generated site shows:
- Table descriptions and column descriptions
- Data lineage DAG (see how models flow from sources to marts)
- Test status
- Column-level lineage

---

## dbt Best Practices

### Naming Conventions

| Type | Prefix | Example |
|------|--------|---------|
| Sources | (none) | Referenced via `source()` |
| Staging | `stg_` | `stg_stripe__payments` |
| Intermediate | `int_` | `int_orders_joined` |
| Fact tables | `fct_` | `fct_orders` |
| Dimension tables | `dim_` | `dim_customers` |
| Mart summaries | (descriptive) | `daily_revenue_summary` |

### Performance Tips

1. **Incremental models for large tables**: Full refresh of 100M row tables is expensive. Use `is_incremental()`.
2. **Partition by date**: `partition_by` in config lets BigQuery/Snowflake prune partitions.
3. **Z-ordering / clustering**: Cluster by your most common filter columns.
4. **Avoid complex CTEs in views**: If a view takes 30s to query, materialize it as a table.
5. **Use `+tags:` to run subsets**: `dbt run --select tag:daily` runs only daily models.

### State-Based CI

In CI, only test and run modified models:
```bash
# Get production artifacts
dbt ls --target prod --output json > prod-manifest.json

# Run only modified models and their downstream dependents
dbt run --select state:modified+ --state prod-artifacts/ --defer
dbt test --select state:modified+ --state prod-artifacts/
```

---

*← [07 - Cloud Platforms](07-cloud-platforms.md) | [09 - Hands-On Course](09-hands-on-course.md) →*
