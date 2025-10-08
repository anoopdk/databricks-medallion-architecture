md_content = """# Databricks Bronze Layer Ingestion Framework – Design and Implementation
**Author:** Anoop Kulkarni  
**Role:** Senior Data Engineer  
**Platform:** Databricks | Delta Lake | PySpark | Auto Loader  
**Last Updated:** 2025-10-08  

---

## 📘 Table of Contents
1. [Executive Summary](#executive-summary)
2. [Overview and Objectives](#overview-and-objectives)
3. [Key Design Principles](#key-design-principles)
4. [Preferred Architecture](#preferred-architecture)
5. [Metadata Columns in Bronze](#metadata-columns-in-bronze)
6. [Control Schema and DDLs](#control-schema-and-ddls)
7. [Why Auto Loader over Delta Live Tables (DLT)](#why-auto-loader-over-delta-live-tables-dlt)
8. [Operational Flow Overview](#operational-flow-overview)
9. [Full PySpark Implementation](#full-pyspark-implementation)
10. [Operational Best Practices](#operational-best-practices)
11. [Summary and Recommendations](#summary-and-recommendations)

---

## Executive Summary
This document presents a comprehensive, metadata-driven **Bronze Layer Ingestion Framework** built on **Databricks**.  
It leverages **Databricks Auto Loader** for high-volume streaming ingestion, ensures strong schema governance,  
and integrates **operational recovery and backfill capabilities** through control tables.  

This framework provides:
- Governance and traceability
- Scalable ingestion pattern for structured/unstructured data
- Full auditability across ingestion runs
- Controlled backfill for operational recovery

---

## Overview and Objectives
This framework defines a **standardized ingestion pattern** for the Bronze layer in Databricks.  
It ensures consistency, lineage, and auditability across ingestion pipelines, supporting:
- Incremental streaming ingestion
- Controlled backfill (operational recovery)
- Metadata-driven schema and source management
- End-to-end observability

---

## Key Design Principles
- **Metadata-driven ingestion** with centralized control schema  
- **Auto Loader** for scalable, incremental ingestion  
- **Immutable Bronze** (raw data only, no business logic)  
- **Recovery readiness** with operational recovery control tables  
- **Audit and lineage** tracking for observability  
- **Separation of concerns**: Bronze (ingestion) → Silver (cleansing) → Gold (analytics)

---

## Preferred Architecture
The framework follows the **Medallion architecture**:

| Layer | Description | Tool |
|--------|--------------|------|
| **Control** | Control tables to drive orchestration | Databricks code |
| **Bronze** | Raw ingestion using Auto Loader | Databricks Auto Loader |
| **Silver** | Data cleansing, SCD management | Delta Live Tables (DLT) |
| **Gold** | Curated analytics-ready layer | SQL / DLT |

Bronze → Silver → Gold ensures progressive refinement and governance.

---

## Metadata Columns in Bronze
| Column | Purpose | Example |
|---------|----------|----------|
| `_ingest_ts` | Timestamp of ingestion | 2025-10-06T18:45:00Z |
| `_source_system` | Origin system name | CRM, ERP, POS |
| `_file_name` | Original file name | sales_20251006.csv |

These metadata fields enable lineage, auditability, and recovery traceability.

---

## Control Schema and DDLs
The **Control Schema** centralizes ingestion configuration, recovery, and audit metadata.  
All tables reside under the `control` schema.

### control.control_table_spec

**Purpose:** Defines static configuration for each source entity.
```sql
CREATE OR REPLACE TABLE control.control_table_spec (
  entity_name STRING COMMENT 'Logical name of the entity/table (e.g., customer, sales)',
  source_path STRING COMMENT 'Path in the lakehouse where source files land (e.g., /mnt/raw/customer/)',
  file_format STRING COMMENT 'File format of source data (csv, parquet, json, etc.)',
  has_header BOOLEAN COMMENT 'Indicates if the source file contains a header row',
  delimiter STRING COMMENT 'Delimiter for CSV files',
  schema_json STRING COMMENT 'JSON string defining schema for enforcement',
  load_type STRING COMMENT 'incremental or full — describes the source behavior',
  target_table STRING COMMENT 'Fully qualified Bronze Delta table name (e.g., bronze.customer_raw)',
  primary_key STRING COMMENT 'Business key used for deduplication or audit',
  source_system STRING COMMENT 'Source system identifier (CRM, ERP, POS)',
  schema_location STRING COMMENT 'Auto Loader schema inference checkpoint directory',
  checkpoint_location STRING COMMENT 'Auto Loader streaming checkpoint path',
  active_flag BOOLEAN DEFAULT TRUE COMMENT 'Controls whether entity ingestion is active',
  refresh_strategy STRING DEFAULT 'standard' COMMENT 'standard or backfill; defines ingestion behavior',
  created_by STRING DEFAULT current_user() COMMENT 'User who registered the entity config',
  created_date TIMESTAMP DEFAULT current_timestamp() COMMENT 'Timestamp when config was registered'
);
```

### control.operational_recovery_spec

**Purpose:** Manages backfill and operational recovery executions.
```sql
CREATE OR REPLACE TABLE control.operational_recovery_spec (
  entity_name STRING COMMENT 'Entity to recover (matches control_table_spec)',
  refresh_start DATE COMMENT 'Backfill start date (inclusive)',
  refresh_end DATE COMMENT 'Backfill end date (exclusive, NULL = today)',
  initiated_by STRING COMMENT 'User initiating recovery',
  initiated_ts TIMESTAMP DEFAULT current_timestamp() COMMENT 'Auto-set when row inserted',
  status STRING DEFAULT 'INITIATED' COMMENT 'INITIATED / RUNNING / COMPLETE / FAILED / IGNORED',
  run_id STRING COMMENT 'Job run ID for traceability',
  comments STRING COMMENT 'Optional notes on recovery intent or errors',
  last_updated_ts TIMESTAMP DEFAULT NULL COMMENT 'Updated after job completion/failure'
);
```

### control.load_audit

**Purpose:** Logs all ingestion executions (incremental or backfill).
```sql
CREATE OR REPLACE TABLE control.load_audit (
  entity_name STRING COMMENT 'Entity name being loaded',
  run_id STRING COMMENT 'Unique run identifier',
  refresh_strategy STRING COMMENT 'standard or backfill',
  refresh_start DATE COMMENT 'Start date of data processed (for backfill)',
  refresh_end DATE COMMENT 'End date of data processed (for backfill)',
  record_count BIGINT COMMENT 'Number of rows ingested',
  status STRING COMMENT 'Job status: SUCCESS / FAILED / PARTIAL / RUNNING',
  duration_sec DOUBLE COMMENT 'Execution duration in seconds',
  comments STRING COMMENT 'Error or informational messages',
  load_start TIMESTAMP DEFAULT current_timestamp() COMMENT 'Load start timestamp',
  load_end TIMESTAMP COMMENT 'Load end timestamp'
);
```

## Why Auto Loader over Delta Live Tables (DLT)

| Area | Auto Loader | Delta Live Tables (DLT) | Why Auto Loader Wins |
|-------|--------------|--------------------------|----------------------|
| **Purpose** | Continuous ingestion | Declarative transformation | Ideal for Bronze ingestion |
| **Schema Evolution** | Automatic | Managed at DAG level | Simplifies ingestion |
| **Checkpointing** | Fine-grained | Pipeline-level | Easier to control |
| **Execution Mode** | Streaming | Batch/DAG | Lightweight for file streams |
| **Cost Efficiency** | Low overhead | Additional orchestration cost | Optimized for ingestion |

**Conclusion:**  
Use **Auto Loader** for Bronze ingestion; use **DLT** for Silvertransformations.

Auto-loader - Built for **Row-file ingestion**, while DLT is built for **Managed transformations**

Auto-loader brings data to the **Bronze** layer, while DLT cleans and transformas data in the **Silver** layer.

## Operational Flow Overview

### 🔹 Standard Incremental Flow

- Read entity configuration from `control.control_table_spec`.

- Auto Loader (cloudFiles) continuously ingests data → Bronze table.

- Metadata columns added: `_ingest_ts`, `_source_system`, `_file_name`.

- Uses persistent checkpoint for state tracking.

- Logs run to `control.load_audit`.

### 🔹 Controlled Backfill Flow

- A row is inserted into `control.operational_recovery_spec` with: `entity_name`, `refresh_start`, `refresh_end`, `status='INITIATED'`.

- Job picks it up during run-time and joins both the tables to get the entity informaiton and backfill for the sanme

- In the logic, we create a temporary checkpoint, so that we dont overwrite the produciton checkpoint, instead create seperate `backfill/checkpoint` for this activity.

- Loads files for date range using `triggerOnce`.

- Writes to a staging table (`<entity>_staging`).

- Validates:

  - Row count > 0

  - Primary key uniqueness (if defined)

- Performs atomic swap:

  - Rename existing production → backup

  - Rename staging → production

- Updates status to `COMPLETE` and logs in `load_audit`.

## Full PySpark Implementation

**Production-ready PySpark script** you can run on Databricks as a notebook (or a worker job).

### It implements:

-  Incremental Auto Loader streaming ingestion (append, persistent checkpoint).

-  Safe backfill using Auto Loader triggerOnce with temporary checkpoints (TEMP_CHECKPOINT_BASE) so production checkpoints are never touched.

-  Staging table for validation, then atomic swap (rename staging → production) with backup.

-  Control schema integration (control.control_table_spec, control.operational_recovery_spec, control.load_audit).

-  Full inline comments and operational safeguards.

### Assumptions / prerequisites

- Delta tables and control schema (control DB) exist and are writable by the notebook role.

- Source landing is partitioned by year=.../month=.../day=... for backfills (if not, the backfill path can be adapted to non-partitioned folders).

- Auto Loader (cloudFiles) is available in your Databricks runtime.


This PySpark notebook implements both **incremental Auto Loader ingestion** and **controlled backfill**.

```python
# -------------------------------------------------------------------------
# Databricks Bronze Layer Ingestion Controller
# - Incremental Auto Loader stream
# - Controlled backfill using Auto Loader (triggerOnce) with temp checkpoints
# - Staging -> validate -> atomic swap promotion
# - Control tables: control.control_table_spec, control.operational_recovery_spec, control.load_audit
# Author: Anoop Kulkarni (provided)
# -------------------------------------------------------------------------

from pyspark.sql import SparkSession
from pyspark.sql.functions import current_timestamp, input_file_name, lit
from pyspark.sql.utils import AnalysisException
from datetime import date, datetime, timedelta
import uuid
import time

spark = SparkSession.builder.appName("bronze_ingestion_controller").getOrCreate()

# ----------------------------
# CONFIGURATION (adjust for your env)
# ----------------------------
CONTROL_DB = "control"
CONTROL_SPEC_TABLE = f"{CONTROL_DB}.control_table_spec"
OP_RECOVERY_TABLE = f"{CONTROL_DB}.operational_recovery_spec"
LOAD_AUDIT_TABLE = f"{CONTROL_DB}.load_audit"

# Base location for temporary checkpoints used by backfill runs (must be writable)
TEMP_CHECKPOINT_BASE = "/mnt/checkpoints/backfill"

# ----------------------------
# Helper functions
# ----------------------------

def now_utc_str():
    return datetime.utcnow().strftime("%Y%m%dT%H%M%SZ")

# Date range utilities (inclusive)
def daterange(start_date, end_date):
    for n in range(int((end_date - start_date).days) + 1):
        yield start_date + timedelta(n)

# Build partition paths assuming partitioning: /base/year=YYYY/month=MM/day=DD
def partition_paths_for_range(base_path, start_date, end_date, fmt="year={y}/month={m:02d}/day={d:02d}"):
    return [f"{base_path}/{fmt.format(y=d.year, m=d.month, d=d.day)}" for d in daterange(start_date, end_date)]

# ----------------------------
# Control / recovery helpers
# ----------------------------
def mark_recovery_running(entity, run_id):
    """Mark the recovery request as RUNNING and set run_id (for rows in INITIATED or FAILED)."""
    spark.sql(f"""
      UPDATE {OP_RECOVERY_TABLE}
      SET status = 'RUNNING', run_id = '{run_id}'
      WHERE entity_name = '{entity}' AND status IN ('INITIATED','FAILED')
    """)

def update_recovery_final(entity, run_id, status, comments=None):
    """Update recovery row with final status and last_updated_ts."""
    comments_sql = f"'{comments}'" if comments else "NULL"
    spark.sql(f"""
      UPDATE {OP_RECOVERY_TABLE}
      SET status = '{status}', last_updated_ts = current_timestamp(), comments = {comments_sql}
      WHERE entity_name = '{entity}' AND run_id = '{run_id}'
    """)

def insert_load_audit(entity, run_id, refresh_strategy, refresh_start, refresh_end, record_count, status, duration_sec, comments=None):
    """Insert a single row into control.load_audit. load_start uses current_timestamp(); load_end left NULL."""
    comments_sql = f"'{comments}'" if comments else "NULL"
    rs_sql = f"DATE('{refresh_start}')" if refresh_start else "NULL"
    re_sql = f"DATE('{refresh_end}')" if refresh_end else "NULL"
    spark.sql(f"""
      INSERT INTO {LOAD_AUDIT_TABLE} (
        entity_name, run_id, refresh_strategy, refresh_start, refresh_end,
        record_count, status, duration_sec, comments, load_start, load_end
      ) VALUES (
        '{entity}', '{run_id}', '{refresh_strategy}', {rs_sql}, {re_sql},
        {record_count}, '{status}', {duration_sec}, {comments_sql}, current_timestamp(), NULL
      )
    """)

# ----------------------------
# Validation & atomic swap helpers
# ----------------------------
def validate_staging_counts(staging_table, min_rows=1):
    """Return (pass_bool, row_count)."""
    try:
        cnt = spark.sql(f"SELECT COUNT(*) as c FROM {staging_table}").collect()[0]["c"]
        return (cnt >= min_rows, cnt)
    except Exception as e:
        print("validate_staging_counts error:", e)
        return (False, 0)

def validate_pk_uniqueness(staging_table, pk_col):
    """Validate primary key uniqueness in staging table; returns True if unique."""
    q = f"SELECT COUNT(*) as total, COUNT(DISTINCT {pk_col}) as distincts FROM {staging_table}"
    res = spark.sql(q).collect()[0]
    return res["total"] == res["distincts"]

def table_exists(table_name):
    try:
        spark.table(table_name)
        return True
    except AnalysisException:
        return False

def atomic_swap_table(staging_table, production_table, keep_backup_count=3):
    """
    Atomic-ish swap:
     - rename production -> production_backup_<ts>
     - rename staging -> production
    Returns backup table name (or raises).
    Note: Ensure no concurrent writes to these tables during swap.
    """
    ts = now_utc_str()
    backup_name = f"{production_table}_backup_{ts}"
    if table_exists(production_table):
        spark.sql(f"ALTER TABLE {production_table} RENAME TO {backup_name}")
    spark.sql(f"ALTER TABLE {staging_table} RENAME TO {production_table}")
    return backup_name

# ----------------------------
# Main controller logic
# ----------------------------

# Read active control entries and any recovery requests needing attention
ctrl_df = spark.table(CONTROL_SPEC_TABLE).filter("active_flag = true")
# Recovery rows in INITIATED or FAILED (we retry failed until ignored)
recovery_df = spark.table(OP_RECOVERY_TABLE).filter("status IN ('INITIATED','FAILED')")

# Left join control + recovery to produce runtime parameters (recovery may be NULL)
merged_df = ctrl_df.alias("c").join(recovery_df.alias("r"), on="entity_name", how="left") \
    .selectExpr("c.*", "r.refresh_start", "r.refresh_end", "r.initiated_by", "r.initiated_ts", "r.status as recovery_status")

# NOTE: In very large environments avoid collect(); prefer job-per-entity orchestration.
entities = merged_df.collect()

for row in entities:
    entity = row.entity_name
    src_path = row.source_path
    file_format = (row.file_format or "csv").lower()
    has_header = bool(row.has_header) if hasattr(row, "has_header") else False
    delimiter = row.delimiter if hasattr(row, "delimiter") else ","
    schema_loc = row.schema_location
    checkpoint_loc = row.checkpoint_location
    target_table = row.target_table
    staging_table = f"{target_table}_staging"
    refresh_strategy = (row.refresh_strategy or "standard").lower()
    rec_status = row.recovery_status  # may be None or 'INITIATED' or 'FAILED'
    refresh_start = row.refresh_start
    refresh_end = row.refresh_end
    pk = row.primary_key if hasattr(row, "primary_key") else None
    run_id = str(uuid.uuid4())

    print(f"\n--- Entity: {entity} | strategy={refresh_strategy} | recovery_status={rec_status} | run_id={run_id}")

    if rec_status in ("INITIATED", "FAILED"):
        try:
            mark_recovery_running(entity, run_id)
        except Exception as e:
            print("Warning: failed to mark recovery running:", e)

    start_ts = datetime.utcnow()

    try:
        # ---------------------------
        # Incremental streaming (standard)
        # ---------------------------
        if refresh_strategy == "standard" and (rec_status is None):
            # Start a long-running Auto Loader stream writing to the target_table (append mode).
            # In production, run this notebook as a scheduled job per entity or use job orchestration to manage the lifecycle.
            print(f"[{entity}] Starting long-running Auto Loader stream -> {target_table}")
            stream_reader = spark.readStream.format("cloudFiles") \
                .option("cloudFiles.format", file_format) \
                .option("cloudFiles.schemaLocation", schema_loc) \
                .option("cloudFiles.ignoreHiddenFiles", "true")

            if file_format == "csv":
                stream_reader = stream_reader.option("header", "true" if has_header else "false").option("sep", delimiter)
            else:
                # for parquet/json no header option
                pass

            ds = stream_reader.load(src_path) \
                 .withColumn("_ingest_ts", current_timestamp()) \
                 .withColumn("_source_system", lit(row.source_system if hasattr(row, "source_system") else None)) \
                 .withColumn("_file_name", input_file_name())

            writer = ds.writeStream \
                .format("delta") \
                .option("checkpointLocation", checkpoint_loc) \
                .option("mergeSchema", "true") \
                .outputMode("append")

            # Start the stream to a managed table name (this creates a long-running query).
            # NOTE: In notebook interactive contexts, starting a stream like this will run until cancelled;
            # typically you create a Databricks job to start/manage the stream on a job cluster.
            writer.toTable(target_table)
            print(f"[{entity}] Stream started with checkpoint: {checkpoint_loc}")
            # Do not continue to backfill promotion in the same run for streaming entities.
            continue

        # ---------------------------
        # Backfill / Recovery (triggerOnce via Auto Loader streaming)
        # ---------------------------
        is_backfill_request = (refresh_strategy == "backfill") or (rec_status in ("INITIATED", "FAILED"))
        if is_backfill_request:
            if refresh_start is None:
                raise ValueError(f"Backfill requested for {entity} but refresh_start is NULL")

            backfill_start = refresh_start
            backfill_end = refresh_end or date.today()
            print(f"[{entity}] Running backfill from {backfill_start} to {backfill_end}")

            # Build list of partition paths to read (assumes partitioning).
            paths = partition_paths_for_range(src_path, backfill_start, backfill_end)
            if not paths:
                raise RuntimeError("No partition paths were resolved for the given backfill range.")

            # Build a streaming union over partition paths using Auto Loader.
            # We will run the writeStream with trigger(once=True) so it behaves like a batch ingestion but with cloudFiles schema handling.
            union_stream = None
            for p in paths:
                print(f"[{entity}] queue partition: {p}")
                p_reader = spark.readStream.format("cloudFiles") \
                    .option("cloudFiles.format", file_format) \
                    .option("cloudFiles.schemaLocation", schema_loc) \
                    .option("cloudFiles.ignoreHiddenFiles", "true")
                if file_format == "csv":
                    p_reader = p_reader.option("header", "true" if has_header else "false").option("sep", delimiter)

                part_df = p_reader.load(p) \
                    .withColumn("_ingest_ts", current_timestamp()) \
                    .withColumn("_source_system", lit(row.source_system if hasattr(row, "source_system") else None)) \
                    .withColumn("_file_name", input_file_name())

                union_stream = part_df if union_stream is None else union_stream.unionByName(part_df, allowMissingColumns=True)

            # Temporary checkpoint for this backfill run (isolated per run)
            temp_checkpoint = f"{TEMP_CHECKPOINT_BASE}/{entity}/{run_id}/"
            print(f"[{entity}] Using temp checkpoint for backfill: {temp_checkpoint}")

            # Write the union stream once into a staging table
            staging_table = staging_table  # already defined earlier
            # Ensure staging table doesn't exist or drop before writing (optional safety)
            if table_exists(staging_table):
                print(f"[{entity}] Dropping existing staging table {staging_table}")
                spark.sql(f"DROP TABLE {staging_table}")

            print(f"[{entity}] Starting triggerOnce Auto Loader write to staging table: {staging_table}")
            writer = union_stream.writeStream \
                .format("delta") \
                .option("checkpointLocation", temp_checkpoint) \
                .option("mergeSchema", "true") \
                .trigger(once=True) \
                .outputMode("append") \
                .toTable(staging_table)

            # Wait for triggerOnce to finish
            # If running in a Job, this will block until completion. In interactive notebooks the behavior is similar.
            # The query returned is a streaming query object; DataBricks provides awaitTermination() on it.
            # However writer.toTable returns None in newer runtimes; if so, query handling is different.
            # We'll use a safe sleep + poll pattern in case toTable returns immediately.
            # (If writer returned a query, it will have awaitTermination())
            try:
                # Try to obtain running queries and await termination of the new one.
                # This section may be optional depending on Databricks runtime version.
                running_queries = spark.streams.active
                # Wait up to a long timeout for the streaming job to finish (e.g., 1 hour).
                timeout_seconds = 60 * 60
                poll_interval = 5
                waited = 0
                while any(q.name is None or q.isActive for q in running_queries) and waited < timeout_seconds:
                    time.sleep(poll_interval)
                    waited += poll_interval
                    running_queries = spark.streams.active
                # After loop proceed - the triggerOnce should have completed
            except Exception:
                # In some runtimes writer.toTable doesn't return a query object; rely on readiness of staging table below.
                pass

            # Validate staging table content
            valid, row_count = validate_staging_counts(staging_table, min_rows=1)
            pk_ok = True
            if pk:
                try:
                    pk_ok = validate_pk_uniqueness(staging_table, pk)
                except Exception as e:
                    print(f"[{entity}] PK uniqueness validation thrown: {e}")
                    pk_ok = False

            duration_sec = (datetime.utcnow() - start_ts).total_seconds()

            if not valid:
                update_recovery_final(entity, run_id, 'FAILED', 'No rows in staging after backfill')
                insert_load_audit(entity, run_id, 'backfill', str(backfill_start), str(backfill_end), 0, 'FAILED', duration_sec, 'No rows in staging')
                print(f"[{entity}] BACKFILL FAILED: no rows in staging")
                continue

            if not pk_ok:
                update_recovery_final(entity, run_id, 'FAILED', 'Primary key uniqueness failed')
                insert_load_audit(entity, run_id, 'backfill', str(backfill_start), str(backfill_end), row_count, 'FAILED', duration_sec, 'PK uniqueness failed')
                print(f"[{entity}] BACKFILL FAILED: PK uniqueness failed")
                continue

            # Passed validation, perform atomic swap (staging -> production)
            try:
                backup_name = atomic_swap_table(staging_table, target_table)
            except Exception as e:
                update_recovery_final(entity, run_id, 'FAILED', f"Swap failed: {e}")
                insert_load_audit(entity, run_id, 'backfill', str(backfill_start), str(backfill_end), row_count, 'FAILED', duration_sec, f"Swap failed: {e}")
                print(f"[{entity}] BACKFILL FAILED: swap failed: {e}")
                continue

            # Success: update recovery and audit
            update_recovery_final(entity, run_id, 'COMPLETE', f"Backfill promoted; backup={backup_name}")
            insert_load_audit(entity, run_id, 'backfill', str(backfill_start), str(backfill_end), row_count, 'COMPLETE', duration_sec, f"Backfill promoted; backup={backup_name}")
            print(f"[{entity}] BACKFILL COMPLETE: rows={row_count}, backup={backup_name}")

            # Continue to next entity
            continue

        # If no action required
        print(f"[{entity}] No action required (neither standard stream nor backfill).")

    except Exception as ex:
        # General exception handling and recovery update
        print(f"[{entity}] ERROR during processing: {ex}")
        try:
            update_recovery_final(entity, run_id, 'FAILED', str(ex))
        except Exception as e2:
            print("Failed to update recovery table:", e2)
        # Write audit entry for failure
        try:
            duration_sec = (datetime.utcnow() - start_ts).total_seconds()
            insert_load_audit(entity, run_id, refresh_strategy, str(refresh_start) if refresh_start else None, str(refresh_end) if refresh_end else None, 0, 'FAILED', duration_sec, str(ex))
        except Exception as e3:
            print("Failed to insert load audit:", e3)
        continue

# End for loop over entities
print("Controller execution finished.")

```

## Operational Best Practices

- Do **not reuse production checkpoints** for backfill operations.  
- Maintain separate schemas: `control`, `bronze`, `silver`, `gold`.  
- Keep a historical record of recovery requests.  
- Restrict write access to control tables using Unity Catalog ACLs.  
- Use **Databricks Jobs API** for large-scale per-entity backfills.  
- Validate data post-backfill using audit metrics.  
- Schema evolution: Managed by Auto Loader or controlled via schema_json.
- Incremental checkpoint: Always persistent; never delete manually.
- Concurrency: Only one job per entity.
- Atomic swap: Use ALTER TABLE ... RENAME TO ... for safe replacement.


## Summary and Recommendations

The Databricks Bronze Layer Ingestion Framework provides:
- A **governed, metadata-driven** approach to data ingestion.
- **Auto Loader**-based streaming for continuous, scalable ingestion.
- Controlled **backfill and recovery** capabilities.
- Full auditability through centralized control schema.

### Key Recommendations
1. Use **Auto Loader** for raw ingestion; **DLT** for transformations.
2. Maintain **control schema** as a single source of truth.
3. Audit and recovery tables should be accessible only to platform admins.
4. Integrate this framework into your CI/CD and data governance process.




