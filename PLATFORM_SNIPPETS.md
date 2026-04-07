# Platform Snippets — João Fonseca (v3)

Data Engineering focus. Azure Lakehouse ETL Platform is the flagship project.
All snippets updated to reflect full four-project portfolio.

---

## A) LinkedIn — About (~1100 chars)

Data Engineer with hands-on delivery across the full modern data stack: Azure Databricks lakehouse, dbt on Snowflake, Python orchestration pipelines, and API ingestion with Airflow.

My flagship project is an end-to-end medallion lakehouse on Azure Databricks — Bronze → Silver → Gold — with a SQL-first Gold layer, a declarative data quality layer (7 checks, queryable PASS/FAIL view), formal model contracts, and ADF orchestration. Executed against a live Databricks SQL Warehouse with run logs and Gold table exports committed to the repository.

Alongside that: a three-layer dbt project on Snowflake with SCD Type 2 snapshots and documented lineage (manifest.json, catalog.json); a Python orchestration platform over PostgreSQL with a production orchestrator, SQL DQ gates, and three GitHub Actions CI workflows; and a GitHub Events ingestion pipeline with S3 partitioned storage, Airflow TaskFlow orchestration, and Athena analytics exposure.

I treat data quality and reproducibility as engineering requirements — not afterthoughts. DQ results are pipeline outputs. Evidence is committed, not described.

Open to Data Engineering, Analytics Engineering, and Data Platform roles — remote.

---

## B) LinkedIn — Headline (single line)

Data Engineer · Azure Databricks · dbt · Snowflake · Python · Airflow · SQL · Lakehouse Architecture · Data Quality

---

## C) Braintrust — Profile Summary

Data Engineer focused on SQL-first data modeling, pipeline reliability, and data quality enforcement.

**Core stack:** Python · Azure Databricks · ADLS Gen2 · Unity Catalog · Delta Lake · Azure Data Factory · dbt Core · Snowflake · PostgreSQL · Apache Airflow · Amazon S3 · Athena · GitHub Actions · Docker

**What I build:**

- Full medallion lakehouses on Azure Databricks: Bronze → Silver → Gold with SQL-driven serving, declarative DQ layers (queryable PASS/FAIL views, committed artifacts), formal model contracts, and ADF orchestration
- Layered dbt projects (staging → intermediate → marts) with Kimball star schema and SCD Type 2 snapshots on Snowflake
- Python orchestration platforms with step registries, gate registries, health check subcommands, and SQL DQ gates over PostgreSQL
- End-to-end API ingestion pipelines: hardened HTTP clients, S3 partitioned storage (Hive-style), Airflow TaskFlow DAGs with contract-based validation, Athena analytics exposure
- Reproducible environments (locked deps, Dockerised setup) with CI workflows gating deployments on quality checks

**Evidence available:** four documented projects with source artefacts, execution artifacts (run logs, DQ exports, manifest.json, Gold table samples), and GitHub Actions CI.

Available for contract and full-time Data Engineering engagements. Remote only.

---

## D) Malt — Short bio

Data Engineer building production pipelines and lakehouse platforms on Azure Databricks, Snowflake, PostgreSQL, and AWS with Python, dbt, and Airflow. Flagship project: SQL-first medallion lakehouse on Azure Databricks with Unity Catalog, Delta Lake, ADF orchestration, declarative DQ layer, and validated execution artifacts. Experienced with layered architectures, SCD Type 2, SQL integrity gates, Airflow orchestration, and GitHub Actions CI. Focused on reproducible, evidence-backed data infrastructure.

---

## E) ARC — Technical Summary

**Role:** Data Engineer / Analytics Engineer

**Stack:** Python · Azure Databricks · ADLS Gen2 · Unity Catalog · Delta Lake · Azure Data Factory · dbt Core · Snowflake · PostgreSQL · Apache Airflow · Amazon S3 · Athena · AWS Glue · SQL · GitHub Actions · Docker · boto3

**Execution record:**

- Built a SQL-first medallion lakehouse on Azure Databricks (Bronze → Silver → Gold): 22 SQL DDL assets executed against Unity Catalog Delta tables via a live Databricks SQL Warehouse (46.6s, run `20260330_222422`, `SUCCESS`). Declarative DQ layer — `dq_summary_v1` (7 checks) and `v_dq_status` (PASS/FAIL view) — runs as part of the pipeline with committed artifacts. Formal model contract (`model_contract_v1`) declares grain and object type for every Gold asset. ADF orchestration: 4 pipelines (master, Bronze, Silver, Gold). CI: GitHub Actions (Databricks bundle schema validation + PySpark smoke tests).

- Built a three-layer dbt project on Snowflake: staging → intermediate → marts. Implemented a Kimball star schema (`dim_customers`, `dim_date`, `fct_orders`, `fct_orders_daily`) and SCD Type 2 via dbt snapshots (`customers_snapshot.sql`, `check` strategy). Enforced data quality with `not_null`, `unique`, `relationships`, and `accepted_values` tests across all layers. Generated dbt Docs with full DAG lineage. Artefacts: `manifest.json`, `catalog.json`, `run_results.json`.

- Built a Python/PostgreSQL analytics platform (PHC Analytics) with a four-layer Medallion pipeline (Bronze → Silver → Gold → Serving). Implemented a production orchestrator with step registry, gate registry, event stream, and health check subcommand. Enforced DQ with SQL zero-row gates (SCD2 temporal integrity, FK validation, uniqueness), a deterministic shell runner (`run_dq_folder.sh`), and three pytest test modules. CI: three GitHub Actions workflows (DQ gate, daily smoke test, governance contract).

- Built a GitHub Events ingestion pipeline: hardened Python HTTP client (10s timeout, 3 retries on 5xx), S3 partitioned storage with Hive-style keys, structured JSON run log written on every ingestion attempt (success and failure paths), Airflow TaskFlow DAG (@daily, task chain: ingest >> validate >> monitor), Athena external tables over raw and log data, layered SQL views (staging → core → mart). CI: GitHub Actions (unit tests + linting + DAG import validation).

---

## F) Applications — Proof Points (copy/paste)

- **Azure Databricks lakehouse (flagship):** SQL-first medallion platform (Bronze → Silver → Gold, 22 SQL assets, Unity Catalog) with declarative DQ layer (`dq_summary_v1`, `v_dq_status`), formal model contracts, ADF orchestration, Python execution harness, and committed execution artifacts (run log, DQ exports, Gold table samples). Validated against a live Databricks SQL Warehouse in 46.6 seconds.

- **dbt + Snowflake dimensional modeling:** Three-layer dbt project (staging → intermediate → marts) targeting Snowflake, implementing a Kimball star schema with `dim_customers`, `dim_date`, `fct_orders`, and SCD Type 2 via dbt snapshots. Documented with dbt Docs (manifest.json, catalog.json, run_results.json available).

- **Data quality as engineering discipline:** SQL zero-row integrity gates (SCD2 temporal consistency, FK validation, uniqueness/not-null); declarative DQ layer in Databricks (`dq_summary_v1` + `v_dq_status`); dbt generic tests (`not_null`, `unique`, `relationships`, `accepted_values`) across all model layers; pytest suites enforcing output schema contracts and parquet partition correctness. Results committed as artifacts, not just logged.

- **Reproducible pipelines + CI:** Deterministic dependency management (uv.lock), Dockerised local environments, Makefile-driven execution. GitHub Actions workflows enforcing data quality gates, daily smoke tests, governance contracts, Databricks bundle schema validation, and unit test suites — all blocking on failure.
