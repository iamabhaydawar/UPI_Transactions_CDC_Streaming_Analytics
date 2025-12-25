# UPI_Transactions_CDC_Streaming_Analytics_Project

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Databricks](https://img.shields.io/badge/Platform-Databricks-orange)](https://databricks.com)
[![Delta Lake](https://img.shields.io/badge/Storage-DeltaLake-lightgrey)](https://delta.io)

## 🚀 Overview

This repository provides a production-ready pattern for implementing a Change Data Capture (CDC) streaming pipeline for UPI (Unified Payments Interface) transactions using Databricks, Delta Lake and Spark Structured Streaming. The pipeline is CDC-aware, supports INSERT/UPDATE/DELETE semantics, and produces hourly merchant aggregations with monitoring and alerting scaffolding.

High-level flow:
```
Mock Data Generator → Raw UPI Transactions (CDC) → Streaming Pipeline → Merchant Aggregations → BI/Dashboards
```

## Table of contents

- Quick start
- Architecture & components
- Data model (tables & sample schema)
- Running the pipeline (notebooks / job)
- CDC processing pattern (important snippets)
- Monitoring & alerting
- Configuration & cluster recommendations
- Troubleshooting
- Contributing & license

---

## Quick start

Prerequisites:
- Databricks workspace with Unity Catalog enabled
- Permissions to create catalogs/schemas/tables
- Python 3.8+ runtime on Databricks cluster
- Delta Lake (Databricks runtime includes this)
- Git checkout of this repo

Execution order (recommended):
1. 01_enhanced_data_model.ipynb — create catalog/schema/tables (enable CDC)
2. 03_enhanced_mock_data_generator.ipynb — start mock data generator (insert/update/delete)
3. 02_realtime_streaming_pipeline.ipynb — start streaming job to process CDC and write aggregations

---

## Architecture & components
<img width="1885" height="937" alt="Screenshot 2025-12-24 194310" src="https://github.com/user-attachments/assets/75908c3a-b0da-4e4a-8ff6-97a9965cb7bb" />

Core pieces:
- Enhanced data model: Delta table(s) with Change Data Feed enabled
- Mock data generator: generates realistic transactions and CDC operations
- Streaming pipeline: Spark Structured Streaming reading Delta change feed
- Aggregations: hourly merchant metrics written to Delta `merchant_aggregations`
- Monitoring & processing_log table: store job metrics and errors

Tech stack:
- Databricks runtime (Spark Structured Streaming)
- Delta Lake (Change Data Feed)
- Unity Catalog (governance)
- Python / PySpark notebooks

---

## Data model

Primary tables:
- raw_upi_transactions_v1 — Raw CDC-enabled transaction table (Delta)
- merchant_aggregations — Hourly aggregated metrics per merchant
- processing_log — Monitoring and processing metadata

Example: Create CDC-enabled Delta table (Unity Catalog)

```sql
CREATE TABLE IF NOT EXISTS upi_analytics.upi_transactions.raw_upi_transactions_v1 (
  transaction_id STRING,
  transaction_timestamp TIMESTAMP,
  transaction_amount DOUBLE,
  status STRING,
  merchant_id STRING,
  merchant_name STRING,
  merchant_category STRING,
  customer_id STRING,
  customer_mobile STRING,
  location_city STRING,
  location_state STRING,
  payment_instrument STRING,
  acquirer_id STRING,
  issuer_id STRING,
  additional_info MAP<STRING,STRING>
)
USING DELTA
PARTITIONED BY (merchant_category)
TBLPROPERTIES (
  'delta.enableChangeDataFeed' = 'true',
  'delta.appendOnly' = 'false'
);
```

Recommended: create a separate compacted table for long-term analytic reads.

Merchant aggregations table (example)

```sql
CREATE TABLE IF NOT EXISTS upi_analytics.upi_transactions.merchant_aggregations (
  agg_hour TIMESTAMP,
  merchant_id STRING,
  merchant_name STRING,
  merchant_category STRING,
  txn_count BIGINT,
  txn_amount DOUBLE,
  success_count BIGINT,
  created_at TIMESTAMP,
  updated_at TIMESTAMP
)
USING DELTA
PARTITIONED BY (date_trunc('day', agg_hour));
```

---

## CDC processing pattern (PySpark snippets)

Read the change data feed (streaming):

```python
raw_changes = (
  spark.readStream
    .format("delta")
    .option("readChangeFeed", "true")
    .option("startingVersion", "latest")  # or startingTimestamp
    .table("upi_analytics.upi_transactions.raw_upi_transactions_v1")
)
```

Normalize CDC operations to a multiplier (+1 insert, -1 delete, 0 update-with-delta handling):

```python
from pyspark.sql.functions import when

changes = raw_changes.withColumn(
    "cdc_multiplier",
    when(col("_change_type") == "insert", 1)
    .when(col("_change_type") == "delete", -1)
    .when(col("_change_type") == "update_postimage", 1)  # track as insert/update logic
    .otherwise(0)
)
```

Aggregate and upsert into merchant_aggregations using Delta MERGE:

```python
aggregated = (
  changes
    .filter("<your validation predicates>")
    .groupBy(window("transaction_timestamp", "1 hour").alias("agg_hour"),
             "merchant_id","merchant_name","merchant_category")
    .agg(
      sum("cdc_multiplier").alias("txn_count"),
      sum(expr("transaction_amount * cdc_multiplier")).alias("txn_amount"),
      sum(when(col("status") == "SUCCESS", col("cdc_multiplier")).otherwise(0)).alias("success_count")
    )
)

# Write incremental results to a staging Delta table, then MERGE into final table
```

MERGE example (SQL or DeltaTable API):

```sql
MERGE INTO upi_analytics.upi_transactions.merchant_aggregations t
USING updates u
ON t.agg_hour = u.agg_hour AND t.merchant_id = u.merchant_id
WHEN MATCHED THEN
  UPDATE SET
    txn_count = t.txn_count + u.txn_count,
    txn_amount = t.txn_amount + u.txn_amount,
    success_count = t.success_count + u.success_count,
    updated_at = current_timestamp()
WHEN NOT MATCHED THEN
  INSERT (agg_hour, merchant_id, merchant_name, merchant_category, txn_count, txn_amount, success_count, created_at, updated_at)
  VALUES (u.agg_hour, u.merchant_id, u.merchant_name, u.merchant_category, u.txn_count, u.txn_amount, u.success_count, current_timestamp(), current_timestamp());
```

Idempotency tips:
- Use the CDC multiplier approach so deletes subtract and inserts add.
- Persist source transaction versions/timestamps in a watermark / checkpoint to avoid double-counting.
- Use MERGE for atomic upserts.

---

## Running the notebooks

Notebooks included:
- 01_enhanced_data_model.ipynb — creates catalog/schema/tables and TBLPROPERTIES for CDC
- 02_realtime_streaming_pipeline.ipynb — streaming job, validation, aggregation, MERGE logic
- 03_enhanced_mock_data_generator.ipynb — mock data generator that issues INSERT/UPDATE/DELETE to the Delta table

Recommended job configuration (Databricks job JSON snippet):

```json
{
  "name": "UPI_Transactions_Real_Time_Streaming",
  "description": "Real-time UPI transactions processing pipeline",
  "max_concurrent_runs": 1,
  "timeout_seconds": 0,
  "tasks": [
    {
      "task_key": "setup-data-model",
      "notebook_task": {"notebook_path": "/path/01_enhanced_data_model"},
      "existing_cluster_id": "<cluster-id>"
    },
    {
      "task_key": "start-streaming",
      "notebook_task": {"notebook_path": "/path/02_realtime_streaming_pipeline"},
      "existing_cluster_id": "<cluster-id>"
    },
    {
      "task_key": "start-mock-generator",
      "notebook_task": {"notebook_path": "/path/03_enhanced_mock_data_generator"},
      "existing_cluster_id": "<cluster-id>"
    }
  ]
}
```

---

## Monitoring & alerting

Tables/queries:
- processing_log table collects run metadata, errors, durations, and processed counts
- SHOW STREAMS; to list active streaming queries
- Example monitoring query:

```sql
SELECT
  DATE_TRUNC('hour', start_time) AS hour,
  SUM(processed_records) as processed_records,
  SUM(error_count) as total_errors,
  AVG(process_duration_seconds) AS avg_duration
FROM upi_analytics.upi_transactions.processing_log
WHERE start_time >= current_timestamp() - INTERVAL 24 HOURS
GROUP BY 1
ORDER BY 1 DESC;
```

Suggested alerts:
- Success rate < 90%
- Any processing job failure
- Streaming query is not active / stopped
- Data quality invalid rate > threshold

Implement alerts via Databricks Jobs notifications, Databricks SQL, or external monitoring (PagerDuty/Slack/Email).

---

## Configuration & cluster recommendations

Environment variables (config file / notebook top):

```python
catalog_name = "upi_analytics"
schema_name = "upi_transactions"
streaming_trigger_interval = "30 seconds"   # or "Trigger.ProcessingTime('30 seconds')" in Spark
monitoring_hours_back = 24
```

Recommended cluster:
- Driver: i3.xlarge
- Workers: i3.xlarge (2-3) — scale by throughput
- Runtime: Databricks Runtime 13.x (or latest stable Spark runtime)
- Use autoscaling with spot instances where cost-sensitive

---

## Troubleshooting

Common steps:
- Check streaming query status: streaming_query.status
- Restart streaming query if stalled:
  - streaming_query.stop(); streaming_query.start()
- Review processing_log for error details
- Validate checkpoint locations and storage permissions
- Optimize partitioning (merchant_category) and file sizes; enable Delta OPTIMIZE if needed

Helpful SQL:

```sql
SHOW STREAMS;
SELECT * FROM upi_analytics.upi_transactions.processing_log ORDER BY start_time DESC LIMIT 50;
```

---

## Contributing

1. Fork the repository
2. Create a feature branch
3. Write code and unit tests (where applicable)
4. Run notebooks locally / on a dev Databricks workspace
5. Open a Pull Request with description and test steps

Please follow repository code style and commit message conventions.

---

## License

This project is licensed under the MIT License 

## Resources

- [Databricks Structured Streaming Guide](https://docs.databricks.com/structured-streaming/index.html)
- [Delta Lake Change Data Feed (CDC)](https://docs.databricks.com/delta/delta-change-data-feed.html)
- [Unity Catalog Overview](https://docs.databricks.com/data-governance/unity-catalog/index.html)

---



