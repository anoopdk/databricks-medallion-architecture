# Silver Layer — Metadata-driven Validation & Audit

**Purpose:** Provide full documentation and production-ready code for a metadata-driven Silver-layer pipeline on Databricks (Delta Lake / PySpark / DLT) that:

- Reads schema and DQ rules from `workspace.control.table_schema_definition` (column-level control table)
- Applies type casting, nullability checks, DQ expectations, and primary-key uniqueness checks
- Routes invalid rows to quarantine with violation flags
- Writes validated data to `silver.<entity>`
- Logs pipeline runs and violations to `silver.pipeline_audit_log` and `silver.rule_violation_log`
- Provides both PySpark job and Delta Live Tables (DLT) implementations

---

## Table of Contents
1. Executive summary
2. Control tables (DDL)
3. Target Delta tables (DDL)
4. High-level architecture
5. Metadata-driven pipeline design
6. PySpark implementation (detailed files)
7. Delta Live Tables implementation (detailed)
8. Quarantine & violation handling
9. Audit, monitoring and alerts
10. Operational best practices
11. CI/CD and deployment notes
12. Testing and validation
13. Appendix: Examples and sample inserts

---

## 1. Executive summary
This document describes a robust, metadata-driven Silver-layer pipeline that enforces schema and data quality using a column-level control table: `workspace.control.table_schema_definition`.

Key features:
- Centralized column-level metadata (type, nullable, expectation, PK flag)
- Generic processing engine that can handle many entities without code changes
- Quarantine and rule violation logging for observability and reprocessing
- Two implementation options: PySpark job (good for custom control) and DLT (declarative, managed)

---

## 2. Control tables (DDL)

### control.table_schema_definition (your provided structure)

```sql
CREATE TABLE IF NOT EXISTS workspace.control.table_schema_definition (
  source_id BIGINT,
  entity_name STRING,
  column_name STRING,
  data_type STRING,
  expectation STRING,
  is_nullable BOOLEAN,
  is_primark_key BOOLEAN,
  description STRING,
  is_active BOOLEAN,
  schema_version INT DEFAULT 1,
  created_by STRING DEFAULT current_user(),
  created_ts TIMESTAMP DEFAULT current_timestamp()
);
```

Notes:
- `data_type` should be a valid Spark SQL type string such as `string`, `long`, `decimal(10,2)`, `date`, `timestamp`, `boolean`.
- `expectation` is a SQL boolean expression relative to the row, e.g. `amount > 0` or `order_date <= current_date()`.
- `is_primark_key` is a boolean indicating column participates in PK.
- `schema_version` lets you version the schema over time.

---

## 3. Target Delta tables (DDL)

### Silver table
For each entity `X` we will produce `silver.X` (typed, validated).

Example generic DDL (created by the job):
```sql
CREATE TABLE IF NOT EXISTS silver.orders (
  order_id STRING,
  order_date DATE,
  amount DECIMAL(10,2),
  -- plus any other typed columns
  _ingest_ts TIMESTAMP,
  _source_file STRING
) USING DELTA;
```

### Quarantine table
```sql
CREATE TABLE IF NOT EXISTS silver.orders_quarantine (
  -- mirror of raw columns (all string or inferred schema) +
  _violation_reason STRING,
  _violation_flags MAP<STRING, BOOLEAN>,
  _ingest_ts TIMESTAMP,
  _source_file STRING
) USING DELTA;
```

### Pipeline audit log
```sql
CREATE TABLE IF NOT EXISTS silver.pipeline_audit_log (
  run_id STRING,
  entity_name STRING,
  source_id BIGINT,
  run_ts TIMESTAMP,
  duration_seconds DOUBLE,
  records_read BIGINT,
  records_valid BIGINT,
  records_invalid BIGINT,
  status STRING,
  comments STRING
) USING DELTA;
```

### Rule violation log
```sql
CREATE TABLE IF NOT EXISTS silver.rule_violation_log (
  run_id STRING,
  entity_name STRING,
  column_name STRING,
  rule STRING,
  violation_count BIGINT,
  sample_values ARRAY<STRING>,
  logged_ts TIMESTAMP DEFAULT current_timestamp()
) USING DELTA;
```

---

## 4. High-level architecture

1. **Control plane**: `workspace.control.table_schema_definition` stores column-level definitions and DQ rules.
2. **Bronze**: Auto Loader lands raw files into `bronze.<entity>_raw` (schema inferred).
3. **Silver processor** (PySpark job or DLT): reads metadata, casts, evaluates rules, writes `silver.<entity>` and `silver.<entity>_quarantine`. Writes logs to `silver.pipeline_audit_log` and `silver.rule_violation_log`.
4. **Monitoring**: Alerting on high invalid rate, repeated violations, or pipeline failures.

---

## 5. Metadata-driven pipeline design

### Key design principles
- **Single source of truth**: column-level metadata drives casting and validation.
- **Immutable Bronze**: Silver never mutates Bronze.
- **Idempotent runs**: Runs can be re-executed and produce consistent outcomes (use run_id and timestamping).
- **Quarantine for remediation**: Invalid rows preserved for analysts to fix.
- **Auditability**: Every run writes summary metrics and detailed violation stats.

### Processing steps (per entity)
1. Read control metadata for `entity_name` and `is_active = true`.
2. Read `bronze.<entity>_raw` (or a time-bound subset) as `bronze_df`.
3. Normalize raw column names (trim, lower) to match metadata column_names (apply mapping if needed).
4. For each metadata column:
   - If raw contains column: cast it to `data_type`.
   - Else: add column with `NULL` of `data_type`.
5. Evaluate nullability constraints: mark rows with NULL in a non-nullable column.
6. Evaluate DQ expectations (SQL expressions) and record boolean success/failure per rule.
7. Check PK uniqueness by grouping on PK columns and marking duplicates.
8. Produce `valid_df` (rows with no violations) and `invalid_df` (rows with any violation). Include violation flags and messages.
9. Persist `valid_df` to `silver.<entity>` (append or merge depending on SCD needs).
10. Persist `invalid_df` to `silver.<entity>_quarantine` (append) with `_violation_flags` map and `_violation_reason` text.
11. Insert run metrics into `silver.pipeline_audit_log`.
12. Compute aggregated violation counts and write to `silver.rule_violation_log`.

---

## 6. PySpark implementation (detailed files)

Below is a suggested repo layout and full code body for key files.

```
repo/
├─ jobs/
│  └─ silver_processor.py   # main job entrypoint
├─ lib/
│  ├─ metadata.py          # helpers to read and shape metadata
│  ├─ validator.py         # core validation logic (casting, expectations, pk checks)
│  ├─ audit.py             # functions to write pipeline and violation logs
│  └─ utils.py             # utility helpers
├─ conf/
│  └─ env.json            # connection and path configs
└─ tests/
   └─ test_validator.py
```

### 6.1 lib/utils.py
```python
from pyspark.sql import functions as F
import uuid

def gen_run_id():
    return str(uuid.uuid4())

def normalize_col_name(c):
    return c.strip().lower()

```

### 6.2 lib/metadata.py
```python
from pyspark.sql import functions as F

def get_entity_metadata(entity_name: str):
    # Read active column-level metadata for entity
    mdf = (
        spark.table("workspace.control.table_schema_definition")
        .filter((F.col("entity_name") == entity_name) & (F.col("is_active") == True))
        .orderBy(F.col("is_primark_key").desc(), F.col("column_name"))
    )
    return mdf

def metadata_to_struct(metadata_df):
    # Return a list of dicts (col_name, data_type, expectation, is_nullable, is_pk)
    rows = metadata_df.collect()
    out = []
    for r in rows:
        out.append({
            "column_name": r["column_name"],
            "data_type": r["data_type"],
            "expectation": r["expectation"],
            "is_nullable": r["is_nullable"],
            "is_pk": r["is_primark_key"]
        })
    return out
```

### 6.3 lib/validator.py
```python
from pyspark.sql import functions as F
from pyspark.sql import Window

def apply_casts_and_defaults(df, metadata):
    # metadata: list of dicts
    for m in metadata:
        col = m["column_name"]
        dtype = m["data_type"]
        if col in df.columns:
            df = df.withColumn(col, F.col(col).cast(dtype))
        else:
            df = df.withColumn(col, F.lit(None).cast(dtype))
    return df


def evaluate_nullability(df, metadata):
    non_nullable = [m["column_name"] for m in metadata if not m["is_nullable"]]
    if non_nullable:
        exprs = [F.col(c).isNull() for c in non_nullable]
        df = df.withColumn("_null_violation", F.reduce(lambda a, b: a | b, exprs))
    else:
        df = df.withColumn("_null_violation", F.lit(False))
    return df


def evaluate_expectations(df, metadata):
    # For each expectation, add a boolean flag column
    for m in metadata:
        col = m["column_name"]
        exp = m.get("expectation")
        if exp and exp.strip() != "":
            # create safe expectation name
            flag_col = f"_dq_{col}_failed"
            # Use expr to evaluate unpredictable SQL boolean
            df = df.withColumn(flag_col, ~F.expr(exp))
    return df


def pk_duplicate_check(df, metadata):
    pk_cols = [m["column_name"] for m in metadata if m.get("is_pk")]
    if pk_cols:
        w = Window.partitionBy(*pk_cols)
        df = df.withColumn("_pk_count", F.count("*").over(w))
        df = df.withColumn("_pk_dup_violation", F.col("_pk_count") > 1)
    else:
        df = df.withColumn("_pk_dup_violation", F.lit(False))
    return df


def gather_violation_flags(df):
    # Collect all internal _ columns that represent violations
    vcols = [c for c in df.columns if c.startswith("_")]
    # Build a map of flag -> boolean for easy storage
    map_expr = F.create_map(*[F.lit(c) for c in vcols], *[F.col(c) for c in vcols])
    df = df.withColumn("_violation_flags", map_expr)
    return df


def split_valid_invalid(df):
    # Any violation flag true means invalid
    vcols = [c for c in df.columns if c.startswith("_")]
    cond = None
    for c in vcols:
        if cond is None:
            cond = F.col(c) == True
        else:
            cond = cond | (F.col(c) == True)
    if cond is None:
        valid_df = df
        invalid_df = df.limit(0).where(F.lit(False))
    else:
        invalid_df = df.filter(cond)
        valid_df = df.filter(~cond)

    # Drop internal _ columns from valid_df
    drop_cols = [c for c in vcols if c not in ("_violation_flags", "_violation_reason")]
    valid_df = valid_df.drop(*drop_cols)
    return valid_df, invalid_df
```

### 6.4 lib/audit.py
```python
from pyspark.sql import functions as F

def log_pipeline_run(run_id, entity_name, source_id, start_ts, end_ts, records_read, records_valid, records_invalid, status, comments=None):
    duration = (end_ts - start_ts).total_seconds()
    spark.createDataFrame([
        (run_id, entity_name, source_id, start_ts, duration, records_read, records_valid, records_invalid, status, comments)
    ], ["run_id","entity_name","source_id","run_ts","duration_seconds","records_read","records_valid","records_invalid","status","comments"]).write.format("delta").mode("append").saveAsTable("silver.pipeline_audit_log")


def log_rule_violations(run_id, entity_name, violation_summary):
    # violation_summary: list of tuples (column_name, rule, count, sample_values)
    rows = []
    for col, rule, cnt, samples in violation_summary:
        rows.append((run_id, entity_name, col, rule, cnt, samples))
    spark.createDataFrame(rows, ["run_id","entity_name","column_name","rule","violation_count","sample_values"]).write.format("delta").mode("append").saveAsTable("silver.rule_violation_log")
```

### 6.5 jobs/silver_processor.py
```python
import time
from datetime import datetime
from lib.utils import gen_run_id
from lib.metadata import get_entity_metadata, metadata_to_struct
from lib.validator import apply_casts_and_defaults, evaluate_nullability, evaluate_expectations, pk_duplicate_check, gather_violation_flags, split_valid_invalid
from lib.audit import log_pipeline_run, log_rule_violations


def process_entity(entity_name, source_id=None):
    run_id = gen_run_id()
    start_ts = datetime.utcnow()
    try:
        meta_df = get_entity_metadata(entity_name)
        metadata = metadata_to_struct(meta_df)

        bronze_df = spark.table(f"bronze.{entity_name}_raw")

        df = apply_casts_and_defaults(bronze_df, metadata)
        df = evaluate_nullability(df, metadata)
        df = evaluate_expectations(df, metadata)
        df = pk_duplicate_check(df, metadata)
        df = gather_violation_flags(df)

        valid_df, invalid_df = split_valid_invalid(df)

        records_read = bronze_df.count()
        records_valid = valid_df.count()
        records_invalid = invalid_df.count()

        # Persist results
        # Upsert/merge logic could be applied to silver table; here we append/overwrite for simplicity
        valid_df.write.format("delta").mode("append").saveAsTable(f"silver.{entity_name}")

        # For quarantine, include violation map and reason
        if records_invalid > 0:
            invalid_df = invalid_df.withColumn("_violation_reason", F.concat_ws(",", *[F.when(F.col(c)==True, F.lit(c)).otherwise(F.lit(None)) for c in invalid_df.columns if c.startswith("_")]))
            invalid_df.write.format("delta").mode("append").saveAsTable(f"silver.{entity_name}_quarantine")

        # Violation summary
        violation_summary = []
        for m in metadata:
            col = m["column_name"]
            flag_col = f"_dq_{col}_failed"
            if flag_col in df.columns:
                cnt = df.filter(F.col(flag_col) == True).count()
                if cnt > 0:
                    samples = df.filter(F.col(flag_col) == True).select(col).limit(5).rdd.flatMap(lambda r: r).collect()
                    violation_summary.append((col, m.get("expectation"), cnt, samples))

        # Log runs and violations
        end_ts = datetime.utcnow()
        status = "SUCCESS"
        log_pipeline_run(run_id, entity_name, source_id, start_ts, end_ts, records_read, records_valid, records_invalid, status)
        if violation_summary:
            log_rule_violations(run_id, entity_name, violation_summary)

        print(f"Run {run_id} complete: {records_valid} valid, {records_invalid} invalid")
    except Exception as e:
        end_ts = datetime.utcnow()
        log_pipeline_run(run_id, entity_name, source_id, start_ts, end_ts, 0, 0, 0, "FAILED", str(e))
        raise

if __name__ == "__main__":
    # Example: process single entity (name can be parameterized via job args)
    process_entity("orders", source_id=101)
```

---

## 7. Delta Live Tables (DLT) implementation
DLT simplifies declarative enforcement and expectations. Below is a DLT pipeline that uses the same control table.

### dlt_silver_pipeline.py
```python
import dlt
from pyspark.sql import functions as F

@dlt.table(name="silver_orders", comment="Validated orders via metadata")
def silver_orders():
    bronze = dlt.read("bronze_orders_raw")

    # Read metadata for this entity
    meta_df = spark.table("workspace.control.table_schema_definition").filter("entity_name = 'orders' AND is_active = true")
    meta = meta_df.collect()

    # Apply casts
    for m in meta:
        col = m["column_name"]
        dtype = m["data_type"]
        if col in bronze.columns:
            bronze = bronze.withColumn(col, F.col(col).cast(dtype))
        else:
            bronze = bronze.withColumn(col, F.lit(None).cast(dtype))

    # Add expectations declaratively
    for m in meta:
        if m["expectation"]:
            dlt.expect_or_drop(f"{m['column_name']}_expect", m["expectation"])

    # DLT will drop rows violating expectations
    # For PK duplicates, you might implement dedupe logic here before returning

    select_cols = [m["column_name"] for m in meta]
    return bronze.select(*select_cols)
```

Notes:
- DLT `expect_or_drop` drops rows failing expectations; if you want to preserve invalid rows, use `expect` with `dlt.write` to separate quarantine.
- DLT provides built-in lineage, metrics, and alerting.

---

## 8. Quarantine & violation handling

- Quarantine keeps original raw payload + violation flags, timestamps, and source file references.
- Maintain retention policy (e.g., retain quarantine for 90 days) and provide analyst workflows to fix and reprocess.
- Quarantine schema should capture both raw fields and metadata about violations (map or string list).

---

## 9. Audit, monitoring and alerts

### pipeline_audit_log
- Use to monitor volume changes, success rates, and SLA violations.
- Alert if `records_invalid / records_read` > threshold (e.g., 5%).

### rule_violation_log
- Aggregated counts per rule; useful for trend detection.
- Create dashboards to show top failing rules and sample bad values.

### Sample monitoring queries
```sql
-- Top entities by invalid rate last 24 hours
SELECT entity_name, SUM(records_invalid) as invalid, SUM(records_read) as read, (SUM(records_invalid)/SUM(records_read)) as invalid_rate
FROM silver.pipeline_audit_log
WHERE run_ts >= current_date() - INTERVAL 1 DAY
GROUP BY entity_name
ORDER BY invalid_rate DESC;
```

### Alerts
- Platform job alert when a run fails.
- Data quality alert when invalid_rate exceeds threshold.

---

## 10. Operational best practices
- **Idempotency**: use `run_id` to dedupe audit logs; for writes to Silver, use `MERGE` when possible to support upserts.
- **Backfills**: use temp checkpoints and separate runs for backfill to avoid interfering with streaming checkpoints.
- **Permissions**: restrict write access to `workspace.control.*` and `silver.*` to platform engineers.
- **Schema changes**: version metadata, and require review when `data_type` or `is_nullable` changes.
- **Testing**: unit tests for validator functions and integration tests with sample Bronze data.
- **Document exceptions**: keep a playbook for common rule failures and remediation steps.

---

## 11. CI/CD and deployment notes
- Package `lib/` as a wheel and install into Job/Cluster init.
- Keep `jobs/silver_processor.py` as the job entrypoint; parameterize entity via job arguments.
- For DLT, maintain pipeline code in repo and deploy using Databricks Repos or via Jobs API.

---

## 12. Testing and validation
- Unit tests for `apply_casts_and_defaults`, expectation evaluation, and PK checks.
- Integration test job: small sample Bronze dataset, expected Silver dataset, and expected quarantine.

Example pytest skeleton (tests/test_validator.py):
```python
from lib.validator import apply_casts_and_defaults

def test_casting_basic(spark):
    df = spark.createDataFrame([("1","2025-10-18","20.5")], ["id","date","amount"])
    metadata = [{"column_name":"id","data_type":"string","is_nullable":False},{"column_name":"date","data_type":"date","is_nullable":True},{"column_name":"amount","data_type":"decimal(10,2)","is_nullable":True}]
    out = apply_casts_and_defaults(df, metadata)
    assert 'amount' in out.schema.names
```

---

## 13. Appendix: Examples and sample inserts

### Insert sample metadata rows
```sql
INSERT INTO workspace.control.table_schema_definition VALUES
(101, 'orders', 'order_id', 'string', 'length(order_id) > 0', false, true, 'Order identifier', true, 1, current_user(), current_timestamp());

INSERT INTO workspace.control.table_schema_definition VALUES
(101, 'orders', 'order_date', 'date', 'order_date <= current_date()', true, false, 'Order date', true, 1, current_user(), current_timestamp());

INSERT INTO workspace.control.table_schema_definition VALUES
(101, 'orders', 'amount', 'decimal(10,2)', 'amount > 0', false, false, 'Order amount', true, 1, current_user(), current_timestamp());
```

### Example Bronze load (simulate)
```python
spark.createDataFrame([
  ("A100","2025-10-18","20.5"),
  ("A101","18-10-2025","N/A"),
  (None, "2025-10-20","15.00")
], ["order_id","order_date","amount"]).write.mode("overwrite").saveAsTable("bronze.orders_raw")
```

---

## Next steps and extensions
- Add automatic schema drift detection and a GitOps workflow for approving metadata changes.
- Provide remediation notebooks that allow analysts to fix quarantine rows and re-ingest.
- Add support for complex nested types (arrays, maps) and JSON schema validation.

---

If you want, I can now:
- Export this repo as runnable notebooks and scripts (zip) in this workspace.
- Generate a DLT pipeline file ready for Databricks (I already included code above).
- Provide a simplified one-click job spec for Databricks Jobs API.

Which of those would you like next?

