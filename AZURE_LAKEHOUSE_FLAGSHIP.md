# Azure Lakehouse ETL Platform — Flagship Evidence

**Stack:** Azure Databricks · ADLS Gen2 (West Europe) · Unity Catalog · Delta Lake · Azure Data Factory · Databricks SQL Warehouse · Python 3.11 · GitHub Actions

**Repo:** `jcsf2020/azure-lakehouse-etl-platform`

---

## 1. Project Pitch

A production-grade data engineering platform on Azure Databricks implementing a full medallion architecture — Bronze → Silver → Gold — for retail and eCommerce analytics. SQL-driven serving layer, explicit data quality, formal model contracts, and validated execution artifacts committed to the repository.

Built to demonstrate the architecture patterns and execution discipline expected in Azure Databricks data engineering roles — with evidence, not descriptions.

**Most relevant for:** Azure Databricks · Delta Lake · Unity Catalog · Databricks SQL · ADLS Gen2 · Azure Data Factory · medallion/lakehouse architecture · data quality implementation · Python data engineering · SQL-first serving layer design

---

## 2. Architecture

```text
Source Systems (CSV / JSON)
         │
         ▼
┌─────────────────────────┐
│  Azure Data Factory     │  Orchestration, scheduling, pipeline control
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│  ADLS Gen2              │  Unified storage — stazlakeetlweu01 (West Europe)
│  Bronze / Silver zones  │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│  Azure Databricks       │  Compute — PySpark (Bronze→Silver) + SQL (Gold)
│  Unity Catalog          │  Namespace: lakehouse_prod.*
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│  Gold Layer (SQL)       │  Facts · Dims · Aggregates · DQ · Contracts
│  Databricks SQL         │  Primary serving surface
└─────────────────────────┘
```

---

## 3. Data Model by Layer

### Bronze — `lakehouse_prod.bronze`

Five raw tables ingested from ADLS Gen2 seed files. Schema inferred via `read_files()`. `_rescued_data` captures malformed fields.

| Table | Source Format | Primary Key |
|-------|---------------|-------------|
| `orders_raw` | JSON | `order_id` |
| `order_items_raw` | JSON | `order_item_id` |
| `customers_raw` | JSON | `customer_id` |
| `products_raw` | JSON | `product_id` |
| `returns_raw` | CSV | `return_id` |

Storage path: `abfss://bronze@stazlakeetlweu01.dfs.core.windows.net/azure-lakehouse-etl/seed/`

### Silver — `lakehouse_prod.silver`

Five cleaned tables. Rows where `_rescued_data IS NOT NULL` are excluded. Natural keys preserved; no surrogate keys at this layer.

| Table | Grain | Transformation |
|-------|-------|----------------|
| `orders_clean` | 1 row per order | Filter malformed, drop rescue column |
| `order_items_clean` | 1 row per order line | Filter malformed, drop rescue column |
| `customers_clean` | 1 row per customer | Filter malformed, drop rescue column |
| `products_clean` | 1 row per product | Filter malformed, drop rescue column |
| `returns_clean` | 1 row per return event | Filter malformed, drop rescue column |

### Gold — `lakehouse_prod.gold`

SQL-driven. All 9 objects defined as SQL DDL in `sql/gold/`, executed in Databricks SQL.

**Fact tables**

| Table | Grain | Source Join |
|-------|-------|-------------|
| `fact_sales` | 1 row per `order_item_id` | INNER JOIN `order_items_clean` + `orders_clean` |
| `fact_returns_enriched` | 1 row per `return_id` | LEFT JOIN `returns_clean` → `fact_sales` |

**Dimension tables** (Type 1 — natural key overwrite)

| Table | Grain |
|-------|-------|
| `dim_customers` | 1 row per `customer_id` |
| `dim_products` | 1 row per `product_id` |

**Aggregation tables**

| Table | Grain | Source |
|-------|-------|--------|
| `orders_channel_summary` | 1 row per channel | `orders_clean` |
| `returns_reason_summary` | 1 row per return reason | `returns_clean` |
| `returns_by_product` | 1 row per product | `fact_returns_enriched` |

**Views**

| View | Description |
|------|-------------|
| `v_sales_enriched` | `fact_sales` + `dim_customers` + `dim_products` context |
| `v_returns_enriched` | `fact_returns_enriched` + `dim_customers` + `dim_products` context |

**Data quality layer**

| Object | Type | Description |
|--------|------|-------------|
| `dq_summary_v1` | TABLE | One row per check: duplicates, nulls, unmatched returns (7 checks) |
| `v_dq_status` | VIEW | PASS / FAIL status per check derived from `dq_summary_v1` |

**Model contract**

| Object | Type | Description |
|--------|------|-------------|
| `model_contract_v1` | TABLE | Declared grain, object type, and layer for all Gold objects |

---

## 4. Validated Runtime

| Item | Detail |
|------|--------|
| Entry point | `scripts/run_full_pipeline.py` |
| Compute | Databricks SQL Warehouse (`databricks-sql-connector`) |
| SQL assets executed | 22 DDL files in dependency order against Unity Catalog |
| Latest run | `20260330_222422` — 46.6 seconds, 17 SQL steps, `SUCCESS` |
| Modes | Full run · `--skip-bronze` · `--dry-run` |
| Parameterization | `PIPELINE_ENV`, `PIPELINE_CATALOG`, `PIPELINE_STORAGE_ACCOUNT` |

Execution evidence committed to repository: `artifacts/execution_runs/20260330_222422/` contains `run_log.json` (per-step status, timing, DQ results), `dq_status.csv`, and Gold table exports.

Curated artifacts in this pack: `artefacts/azure_lakehouse/`

---

## 5. Data Quality Layer

`dq_summary_v1` is a Delta table — a UNION ALL of 7 checks covering nulls, duplicates, and referential integrity. `v_dq_status` maps each check to PASS or FAIL. Both run as part of the pipeline and their results are committed as artifacts.

**DQ results — run `20260330_222422`:**

| Check | Issue Count | Status |
|-------|-------------|--------|
| `dim_customers_duplicate_customer_id` | 0 | PASS |
| `dim_customers_null_customer_id` | 0 | PASS |
| `dim_products_duplicate_product_id` | 0 | PASS |
| `dim_products_null_product_id` | 0 | PASS |
| `fact_sales_duplicate_order_item_id` | 0 | PASS |
| `fact_sales_null_order_item_id` | 0 | PASS |
| `fact_returns_unmatched_sales` | 3 | FAIL *(expected — LEFT JOIN by design)* |

The `fact_returns_unmatched_sales` FAIL is intentional: `fact_returns_enriched` uses a LEFT JOIN to preserve returns that have no matching sale record. This is documented design behaviour, not a defect.

---

## 6. ADF Orchestration

Four pipelines defined in `adf/pipelines/` (JSON):

| Pipeline | Role |
|----------|------|
| `pl_orchestrate_lakehouse` | Master — sequential Bronze → Silver → Gold |
| `pl_bronze_ingestion` | Triggers Databricks Bronze ingestion notebook |
| `pl_silver_transformations` | Triggers Silver notebooks (parallel, with dependency ordering) |
| `pl_gold_aggregations` | Triggers Databricks Gold SQL job |

Silver order items depend on Silver orders and Silver products completing first. All other Silver jobs run in parallel.

This is the intended production deployment path: ADF schedules and sequences the pipeline; Databricks handles the compute. The SQL assets in `sql/gold/`, `sql/dq/`, and `sql/contracts/` are shared between the ADF path and the Python execution harness.

---

## 7. Python Execution Harness

`scripts/run_full_pipeline.py` — validated execution path for Databricks SQL Warehouse runs. Connects via `databricks-sql-connector` and executes all SQL layers in dependency order.

```bash
python scripts/run_full_pipeline.py               # Full run — all layers
python scripts/run_full_pipeline.py --skip-bronze # Silver onward — Bronze already populated
python scripts/run_full_pipeline.py --dry-run     # Validate asset discovery without executing
```

Environment parameterization:

| Variable | Purpose | Default |
|----------|---------|---------|
| `PIPELINE_ENV` | Named config block from `config/environments.yml` (`dev` / `prod`) | `prod` |
| `PIPELINE_CATALOG` | Override Unity Catalog | `lakehouse_prod` |
| `PIPELINE_STORAGE_ACCOUNT` | Override ADLS Gen2 storage account | `stazlakeetlweu01` |

Per-run artifact output: `artifacts/execution_runs/<run_id>/run_log.json` (per-step status, duration, DQ check results, errors) + CSV exports of key Gold tables.

---

## 8. CI

GitHub Actions (`ci/workflows/`):

- **Databricks bundle schema validation** — validates `databricks.yml` against the Databricks Asset Bundle schema without requiring live Databricks credentials
- **Python smoke tests** — validates the PySpark transformation library (`src/azure_lakehouse_etl/`) used for the ADF path Bronze → Silver notebooks

---

## 9. Repository Structure

```text
azure-lakehouse-etl-platform/
├── sql/
│   ├── bronze/          # 5 raw ingestion scripts (read_files → Delta)
│   ├── silver/          # 5 cleansed table scripts (CTAS from Bronze)
│   ├── gold/            # 9 objects: facts, dims, aggregates, views
│   ├── dq/              # dq_summary_v1 + v_dq_status
│   └── contracts/       # model_contract_v1
│
├── scripts/
│   └── run_full_pipeline.py   # Validated execution harness (SQL-first runtime)
│
├── artifacts/
│   └── execution_runs/        # Timestamped run logs + CSV exports (latest committed)
│
├── adf/
│   └── pipelines/             # 4 ADF pipeline definitions (JSON) — production path
│
├── src/
│   └── azure_lakehouse_etl/   # PySpark transformation library (Bronze→Silver, ADF path)
│
├── databricks/
│   └── notebooks/             # Thin notebook entrypoints (ADF path)
│
├── tests/
│   └── quality/               # CI smoke tests for transformation library
│
├── config/                    # environments.yml (dev/prod config)
└── docs/                      # Architecture, execution model, data model, runbook
```

---

## 10. Platform Signals

| Signal | Detail |
|--------|--------|
| SQL-driven Gold layer | All Gold objects defined as SQL DDL — no Python at serving layer |
| Unity Catalog integration | All objects registered under `lakehouse_prod.*` namespace |
| Declarative DQ layer | `dq_summary_v1` + `v_dq_status` — queryable Delta tables, not just log output |
| Formal model contract | `model_contract_v1` declares grain and object type for every Gold asset |
| Intentional LEFT JOIN semantics | `fact_returns_enriched` preserves unmatched returns by design |
| Azure-native orchestration | ADF controls scheduling, sequencing, and failure handling |
| Two coherent execution paths | SQL-first validated path + ADF + PySpark notebooks production path |
| Separation of concerns | ADF orchestrates; Databricks transforms; Python does not live in Gold |
| Validated execution artifacts | Run logs, DQ CSV, Gold table samples committed to repository |

---

## 11. Known Limitations

Documented honestly. None of these are obscured by the evidence above.

| Limitation | Notes |
|------------|-------|
| Seed data only | Not production scale or traffic; timing reflects small-scale execution |
| Full rebuild semantics | `CREATE OR REPLACE` — no `MERGE`, no incremental watermarks, no CDC |
| No SCD Type 2 | Dimensions use Type 1 overwrite |
| No surrogate keys | Natural keys preserved throughout all layers |
| No live ADF deployment in CI | Pipeline JSON definitions exist; no deployed trigger in the CI boundary |
| No IaC | Workspace, cluster, ADLS provisioning not automated |
| Semi-hardcoded catalog | `{{catalog}}` resolved at runtime; full multi-tenant parameterization not implemented |

---

## 12. Career Use

### CV Bullets

Select 2–4 per application. Lead with the signal most relevant to the target role.

**Core platform:**
- Designed and validated a SQL-first medallion lakehouse on Azure Databricks (Bronze → Silver → Gold), executing 22 SQL DDL assets against Unity Catalog Delta tables in 47 seconds end-to-end against a live SQL Warehouse.
- Implemented a declarative data quality layer — `dq_summary_v1` (7 checks) and `v_dq_status` (PASS/FAIL view) — executed as part of the pipeline with committed artifacts, not post-hoc validation.
- Authored a formal model contract (`model_contract_v1`) declaring grain and object type for all Gold layer assets; a queryable Delta table, not documentation.

**Orchestration and execution:**
- Built Python execution harness (`run_full_pipeline.py`) for Databricks SQL Warehouse pipeline runs — environment parameterization (`dev`/`prod`), dependency-ordered SQL execution, structured JSON run logging, and CSV artifact export.
- Designed Azure Data Factory orchestration (4 pipelines) for medallion lakehouse sequencing: Bronze ingestion → parallel Silver transformation with dependency ordering → SQL-driven Gold build.

**Compact option (space-constrained CVs):**
- Built Azure Databricks lakehouse platform (Bronze → Silver → Gold, 22 SQL assets, Unity Catalog) with SQL-first Gold layer, declarative DQ, model contracts, ADF orchestration, and validated execution artifacts committed to repository.

---

### 60-Second Interview Pitch

> "I built an end-to-end lakehouse ETL platform on Azure Databricks — full medallion architecture, Bronze through Gold, with SQL-driven serving, an explicit data quality layer, and formal model contracts.
>
> The key architectural decision was keeping Gold entirely in SQL DDL — no Python at the serving layer. It keeps the Gold layer declarative, reviewable, and separate from transformation logic.
>
> I built a Python execution harness that runs all 22 SQL assets against a live SQL Warehouse in dependency order, validates the DQ results, and exports structured artifacts per run. The whole thing completes in under 60 seconds on seed data, and the run logs, DQ status, and table samples are committed to the repository — so the evidence is verifiable.
>
> It's a reference platform, not production — seed data, small scale — but it demonstrates the delivery approach and technical discipline I'd bring to an Azure data engineering engagement."

---

### Recruiter Screen Summary

**For: Azure / Databricks data engineering contract roles**

This project demonstrates end-to-end data engineering on the Azure Databricks stack:

- Full medallion lakehouse (Bronze → Silver → Gold) on Azure Databricks and ADLS Gen2
- SQL-driven serving layer via Databricks SQL Warehouse and Unity Catalog — not just PySpark notebooks
- Declarative data quality layer (7 checks, queryable PASS/FAIL view, committed execution artifacts)
- Formal model contracts (grain and object type declared for every Gold asset)
- Azure Data Factory orchestration (4 pipelines — master, Bronze, Silver, Gold)
- Python execution harness with environment parameterization and structured run logging
- GitHub Actions CI (bundle schema validation + transformation library smoke tests)

Execution validated against a live Databricks SQL Warehouse. Evidence is in the repository: structured run logs, DQ status exports, and Gold table samples.
