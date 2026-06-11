---
name: bigquery-analyst
description: >
  Write, run, optimize, and explain BigQuery SQL queries. Use when the user wants to
  query BigQuery datasets, analyze GCP billing data, build data pipelines, schedule
  queries, create views or materialized views, optimize query costs, export results,
  or work with BigQuery ML. Triggers on "BigQuery", "BQ", "bq query", "dataset",
  "data warehouse", "analytics", or "SQL on GCP".
license: Apache-2.0
compatibility: Requires gcloud CLI or bq CLI; optionally Python with google-cloud-bigquery
metadata:
  author: google-agent-skills
  version: "1.0"
  gcp-services: BigQuery, BigQuery ML, Data Transfer Service
---

# BigQuery Analyst

## Quick Start

```bash
# Run a query from CLI
bq query --use_legacy_sql=false \
  'SELECT name, COUNT(*) as cnt FROM `project.dataset.table` GROUP BY name LIMIT 10'

# Run query and save results
bq query --use_legacy_sql=false \
  --destination_table=project:dataset.results \
  --replace \
  'SELECT * FROM `project.dataset.source` WHERE date > "2024-01-01"'
```

## Standard SQL Patterns

### Aggregation & Window Functions

```sql
-- Daily active users with 7-day rolling average
SELECT
  event_date,
  COUNT(DISTINCT user_id) AS dau,
  AVG(COUNT(DISTINCT user_id)) OVER (
    ORDER BY event_date
    ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
  ) AS rolling_7d_avg
FROM `project.analytics.events`
GROUP BY event_date
ORDER BY event_date;
```

### Partitioned Table Queries (Cost Optimization)

```sql
-- ALWAYS filter on partition column to avoid full table scans
SELECT *
FROM `project.dataset.events`
WHERE _PARTITIONDATE BETWEEN '2024-01-01' AND '2024-01-31'
  AND user_id = '12345';
```

### Unnesting Arrays

```sql
-- Flatten repeated records
SELECT
  user_id,
  item.product_id,
  item.quantity
FROM `project.dataset.orders`,
UNNEST(items) AS item
WHERE order_date = CURRENT_DATE();
```

### GCP Billing Analysis

```sql
-- Top 10 most expensive services last 30 days
SELECT
  service.description AS service,
  SUM(cost) AS total_cost,
  SUM(cost) / (SELECT SUM(cost) FROM `project.billing.gcp_billing_export_v1_*`
    WHERE DATE(_PARTITIONTIME) >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)) * 100 AS pct
FROM `project.billing.gcp_billing_export_v1_*`
WHERE DATE(_PARTITIONTIME) >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY service
ORDER BY total_cost DESC
LIMIT 10;
```

## Dataset & Table Management

```bash
# Create dataset
bq mk --dataset --location=US project:my_dataset

# Create table from schema JSON
bq mk --table project:dataset.my_table schema.json

# Load CSV
bq load --source_format=CSV \
  project:dataset.table \
  gs://my-bucket/data.csv \
  name:STRING,age:INTEGER,signup_date:DATE

# Load Parquet (schema auto-detected)
bq load --source_format=PARQUET \
  --autodetect \
  project:dataset.table \
  gs://my-bucket/data/*.parquet

# Export to GCS
bq extract \
  --destination_format=CSV \
  project:dataset.table \
  gs://my-bucket/export/data_*.csv
```

## Scheduled Queries

```bash
# Create scheduled query (runs daily at 9am UTC)
bq mk \
  --transfer_config \
  --project_id=my-project \
  --data_source=scheduled_query \
  --display_name="Daily Summary" \
  --schedule="every 24 hours" \
  --params='{"query":"INSERT INTO `project.dataset.daily_summary` SELECT DATE(event_time) as day, COUNT(*) as events FROM `project.dataset.raw_events` WHERE DATE(event_time) = DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY) GROUP BY 1","destination_table_name_template":"daily_summary","write_disposition":"WRITE_APPEND"}'
```

## Views & Materialized Views

```sql
-- Create view
CREATE OR REPLACE VIEW `project.dataset.active_users` AS
SELECT user_id, MAX(event_date) AS last_seen
FROM `project.dataset.events`
GROUP BY user_id;

-- Create materialized view (refreshes automatically)
CREATE MATERIALIZED VIEW `project.dataset.daily_revenue`
PARTITION BY date
AS
SELECT
  DATE(created_at) AS date,
  SUM(amount) AS revenue
FROM `project.dataset.orders`
GROUP BY date;
```

## Python SDK

```python
from google.cloud import bigquery

client = bigquery.Client(project="my-project")

# Run query
query = """
  SELECT name, COUNT(*) AS cnt
  FROM `my-project.my_dataset.my_table`
  GROUP BY name
  ORDER BY cnt DESC
  LIMIT 5
"""
df = client.query(query).to_dataframe()
print(df)

# Load from DataFrame
job_config = bigquery.LoadJobConfig(
    write_disposition="WRITE_TRUNCATE",
    autodetect=True,
)
job = client.load_table_from_dataframe(df, "my-project.dataset.table", job_config=job_config)
job.result()  # Wait for completion
```

## Cost Optimization Tips

1. **Always partition large tables** by date column — queries touching partitioned columns skip unneeded data.
2. **Cluster tables** on frequently filtered columns (`CLUSTER BY user_id, country`).
3. **Use `SELECT specific_columns`** instead of `SELECT *` — BigQuery charges by bytes scanned.
4. **Preview bytes** before running: check bottom-right of the BQ console, or use `--dry_run` flag.
5. **Cache results** — repeated identical queries within 24h are free.

```bash
# Estimate query cost before running
bq query --use_legacy_sql=false --dry_run \
  'SELECT * FROM `project.dataset.large_table` WHERE date = "2024-01-01"'
```

## Common Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `Table not found` | Wrong project/dataset path | Use full path: `` `project.dataset.table` `` |
| `Quota exceeded` | Too many concurrent queries | Use slots reservation or reduce concurrency |
| `Resources exceeded` | Query too complex | Break into CTEs, add filters, use materialized views |
| `Access denied` | Missing IAM role | Grant `roles/bigquery.dataViewer` or `roles/bigquery.user` |
