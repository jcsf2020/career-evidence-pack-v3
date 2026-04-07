# GitHub Events Data Pipeline — Technical Evidence

*Pack context: supporting project. Flagship project is Azure Lakehouse ETL Platform — see `AZURE_LAKEHOUSE_FLAGSHIP.md`.*

**Repo root:** `~/projects/github-events-data-pipeline`
**Evidence artefacts:** `~/career_evidence/github_events_data_pipeline/artefacts/`

---

## Executive Summary

This project implements a production-oriented ingestion and analytics pipeline over the GitHub Public Events API.

It demonstrates end-to-end data engineering capabilities:
- API ingestion with retry, timeout, and error handling
- partitioned storage in Amazon S3 using Hive-style layout
- run-level observability through structured pipeline metadata
- orchestration with Apache Airflow (TaskFlow API)
- analytical exposure via Amazon Athena external tables and layered views

The system is designed around reliability and traceability:
every run produces a structured log, validation enforces data contracts, and monitoring tasks ensure successful ingestion and non-empty datasets.

This project complements the broader portfolio by covering the ingestion and orchestration layer of the data stack, alongside dbt/Snowflake modeling and ERP analytics pipelines.

---

## 1. Architecture Overview

The pipeline follows a five-layer Medallion pattern implemented across Python, S3, and Athena.

```
GitHub Public Events API
        │
        ▼
[ingestion layer — Python]
  hardened HTTP client → fetch_github_events()
        │
        ▼
[raw zone — Amazon S3]
  s3://joaofonseca-data-platform/raw/github_events/
  Hive-style partitions: year=YYYY/month=MM/day=DD/
  Run metadata: metadata/run_logs/run_{run_id}.json
        │
        ▼
[infrastructure — Athena / AWS Glue Catalog]
  github_events_partitioned   (external table, partitioned)
  pipeline_run_log            (external table, flat)
        │
        ▼
[staging → core → marts — Athena views]
  github_events_partitioned_staging
  github_events_partitioned_core
  mart_events_by_type
        │
        ▼
[orchestration — Apache Airflow TaskFlow API]
  DAG: github_events_ingestion  (@daily, catchup=False)
  ingest_github_events >> validate_run_log_contract >> run_monitoring_check
```

**Technology stack:** Python 3.12 · boto3 · Amazon S3 · Amazon Athena · AWS Glue Catalog · Apache Airflow (TaskFlow API) · Pytest · GitHub Actions · uv / ruff

---

## 2. Ingestion Layer

**Artefacts:** `artefacts/ingestion/`

### 2.1 API client — `github_api.py`

Hardened HTTP client for the GitHub public events endpoint.

- URL is externalized via `config.py` (`GITHUB_EVENTS_URL`, default `https://api.github.com/events`)
- `REQUEST_TIMEOUT_SECONDS = 10` applied to every request
- `MAX_RETRIES = 3` with retry logic scoped to transient 5xx status codes: `{500, 502, 503, 504}`
- Non-retryable 4xx errors raise immediately; exhausted retries raise with last status code

### 2.2 Ingestion orchestration — `github_api_ingestion.py`

Coordinates the fetch → upload → log_run sequence.

- UTC timestamp used for both the S3 key and the run log `run_id`
- S3 key pattern: `{prefix}/year={YYYY}/month={MM}/day={DD}/events_{YYYYMMDD_HHMMSS}.json`
- Payload serialized as newline-delimited JSON (`\n`.join of individual event objects)
- On success: calls `log_success()` with `run_id`, `run_ts`, `partition_path`, `records_ingested`
- On failure: calls `log_failure()` before re-raising the exception — run log is always written

### 2.3 Config — `config.py`

All runtime parameters externalized via environment variables with safe defaults:

| Variable | Default |
|---|---|
| `S3_BUCKET` | `joaofonseca-data-platform` |
| `S3_PREFIX` | `raw/github_events` |
| `GITHUB_EVENTS_URL` | `https://api.github.com/events` |

### 2.4 Run logger — `run_logger.py`

Writes a structured JSON run log to S3 after every ingestion attempt.

Run log schema (both success and failure paths):

| Field | Type | Notes |
|---|---|---|
| `run_id` | string | UTC timestamp `YYYYMMDD_HHMMSS` |
| `run_ts` | string | ISO 8601 UTC datetime |
| `partition_path` | string \| null | S3 key of the data file; null on failure |
| `records_ingested` | int | 0 on failure |
| `status` | string | `SUCCESS` or `FAILED` |
| `error_message` | string \| null | null on success |

S3 location: `metadata/run_logs/run_{run_id}.json`

---

## 3. Validation & Monitoring

**Artefacts:** `artefacts/validation/`, `artefacts/monitoring/`

Two independent post-ingestion checks are run as separate Airflow tasks downstream of ingestion.

### 3.1 Run log contract validation — `validate_run_log.py`

Loads the latest run log from S3 and asserts the contract:

1. All six required fields are present: `run_id`, `run_ts`, `partition_path`, `records_ingested`, `status`, `error_message`
2. `status == "SUCCESS"` — fails the task if the ingestion run itself failed
3. `records_ingested > 0` — fails the task if zero events were written

Raises `ValueError` on any contract violation; the Airflow task fails and surfaces the specific message.

### 3.2 Monitoring check — `check_latest_run.py`

Independently loads and re-checks the latest run log:

1. `status == "SUCCESS"`
2. `records_ingested > 0`

Designed as a standalone operational check decoupled from the contract validation task.

**Known gap:** Neither check enforces a freshness SLA (e.g., "latest run must be within the last 25 hours"). Timestamp-based staleness detection is not yet implemented.

---

## 4. Orchestration

**Artefact:** `artefacts/orchestration/github_events_dag.py`

DAG defined using the Airflow 3 TaskFlow API (`airflow.sdk` — `@dag`, `@task` decorators).

| Property | Value |
|---|---|
| `dag_id` | `github_events_ingestion` |
| `schedule` | `@daily` |
| `catchup` | `False` |
| `retries` | 2 |
| `retry_delay` | 5 minutes |
| `ingest timeout` | 10 minutes |
| `validate/monitor timeout` | 5 minutes each |

Task dependency chain: `ingest_github_events >> validate_run_log_contract >> run_monitoring_check`

Each task imports its entry-point module at runtime, keeping the DAG file import-safe.

---

## 5. Athena / Glue Infrastructure

**Artefacts:** `artefacts/sql/infrastructure/`

### 5.1 Raw events external table — `github_events_raw_external_table.sql`

```sql
CREATE EXTERNAL TABLE IF NOT EXISTS github_events_partitioned (
    id string, type string,
    actor struct<login:string>, repo struct<name:string>,
    created_at string
)
PARTITIONED BY (year string, month string, day string)
ROW FORMAT SERDE 'org.openx.data.jsonserde.JsonSerDe'
WITH SERDEPROPERTIES ('ignore.malformed.json' = 'true')
STORED AS TEXTFILE
LOCATION 's3://joaofonseca-data-platform/raw/github_events'
```

Hive-style partition columns align exactly with the S3 key structure written by the ingestion layer.

### 5.2 Run log external table — `pipeline_run_log_external_table.sql`

```sql
CREATE EXTERNAL TABLE IF NOT EXISTS pipeline_run_log (
    run_id string, run_ts string, partition_path string,
    records_ingested bigint, status string
)
ROW FORMAT SERDE 'org.openx.data.jsonserde.JsonSerDe'
LOCATION 's3://joaofonseca-data-platform/metadata/run_logs'
```

Exposes the operational run log in Athena for SQL-based pipeline observability.

---

## 6. SQL Analytical Layer

**Artefacts:** `artefacts/sql/staging/`, `artefacts/sql/core/`, `artefacts/sql/marts/`, `artefacts/sql/tests/`

### 6.1 Staging view — `github_events_staging.sql`

Flattens the nested Athena structs from the partitioned external table:

- `actor.login` → `actor_login`
- `repo.name` → `repo_name`
- Partition columns (`year`, `month`, `day`) preserved for downstream filtering

Source: `github_events_partitioned`

### 6.2 Core view — `github_events_core.sql`

Applies typing to the staging output:

- `CAST(id AS bigint)` → `event_id`
- `CAST(from_iso8601_timestamp(created_at) AS timestamp)` → `event_ts`
- Partition columns carried through

Source: `github_events_partitioned_staging`

### 6.3 Mart — `mart_events_by_type.sql`

```sql
CREATE OR REPLACE VIEW mart_events_by_type AS
SELECT event_type, COUNT(*) AS total_events
FROM github_events_core
GROUP BY event_type;
```

**Known gap:** This mart references `github_events_core`, not `github_events_partitioned_core`. The partitioned lineage uses the `_partitioned_` naming convention in staging and core; `mart_events_by_type` references a shorter alias. SQL coverage alignment to the live partitioned design is ongoing.

### 6.4 SQL test suite

Three SQL test files in `sql/tests/` covering:

| File | What it checks |
|---|---|
| `core_data_quality.sql` | Total row count, null checks on `event_id`/`event_type`/`event_ts`, duplicate `event_id` check, `MAX(event_ts)` freshness probe — all against `github_events_partitioned_core` |
| `github_events_core_quality.sql` | Same null/duplicate/distribution checks against `github_events_core` |
| `pipeline_run_monitoring.sql` | Latest `run_ts`, run counts by status, full run history — against `pipeline_run_log` |

SQL tests are diagnostic queries; any returned rows from the duplicate check constitute a data quality failure.

---

## 7. Python Test Suite & CI

**Artefacts:** `artefacts/tests/`, `artefacts/ci/python-tests.yml`

Four pytest test files covering:

| File | Scope |
|---|---|
| `test_config.py` | Config parsing and environment variable overrides |
| `test_github_api.py` | API client retry logic, timeout behavior, status code handling |
| `test_check_latest_run.py` | Monitoring check status/row-count assertions |
| `test_validate_run_log.py` | Validation contract field presence and value assertions |

GitHub Actions workflow (`python-tests.yml`) runs on push to `main`, `feat/**`, `chore/**` and on PRs to `main`:

| Step | Command |
|---|---|
| Install dependencies | `uv pip install -r requirements.txt` + pytest, ruff, apache-airflow |
| Unit tests | `python -m pytest tests` |
| Linting | `ruff check .` |
| DAG import validation | `python -c "from orchestration.github_events_dag import dag"` |

The DAG import validation step confirms the DAG file is parseable and the `@dag`-decorated function returns a valid DAG object at import time.

---

## 8. Known Gaps

Documented honestly; none of these are hidden by the evidence above.

| Gap | Notes |
|---|---|
| No IaC | S3 bucket and Glue/Athena objects are live but created manually; no Terraform or CloudFormation yet |
| No deduplication | Ingestion writes new files per run with no idempotency or dedup strategy; duplicate `event_id` values are possible |
| No freshness SLA enforcement | Monitoring checks pass/fail on status and row count but do not enforce a maximum staleness window |
| SQL naming inconsistency | `mart_events_by_type` references `github_events_core` rather than `github_events_partitioned_core`; repo SQL coverage of all live Athena objects is still being aligned |
| Partial Python test coverage | Not all ingestion paths have unit test coverage (e.g., `upload_raw_events`, `log_failure`) |
