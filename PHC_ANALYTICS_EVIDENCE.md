# PHC Analytics Platform — Technical Evidence

*Pack context: supporting project. Flagship project is Azure Lakehouse ETL Platform — see `AZURE_LAKEHOUSE_FLAGSHIP.md`.*

---

## 1. Architecture Overview

**Repo root:** `/Users/joaofonseca/projects/jweb-phc-analytics/deliverables/PHC_Analytics`

The project implements a four-layer Medallion / Kimball architecture targeting PostgreSQL (local and Azure-hosted), with a Streamlit dashboard as the BI serving layer:

```
Source (Bronze) → Normalize (Silver) → Model (Gold) → Aggregates → Outputs / BI
     ↓                  ↓                   ↓              ↓
Raw payloads      Validated +         Star schema     Precomputed
 (mock/API)       standardised         (Kimball)       metrics
```

- **Bronze / Extract:** `src/phc_analytics/integrations/` — PrestaShop mock client; Odoo stubs. Raw payloads fetched via `PrestaShopClient`.
- **Silver / Normalize:** `src/phc_analytics/transformations/prestashop_normalize.py` — type-coercion, validation, standardisation of customers, products, orders, order lines.
- **Gold / Model:** `src/phc_analytics/transformations/dim_*.py`, `fact_*_enrich.py` — dimensional schema construction. SQL assets in `sql/analytics/`.
- **Serving / Aggregates:** `src/phc_analytics/transformations/agg_sales_by_product.py` — pre-computed metrics written to `out/`. Dashboard served via `app.py` (Streamlit).

Key components detected:
- `orchestration/` — production orchestrator with step registry, gate registry, event stream, and health check
- `sql/analytics/` — SQL dimensions, facts, aggregates, and numbered DQ checks
- `sql/ci/` — CI seed scripts and governance contract
- `tests/` — pytest test suite (3 test files)
- `.github/workflows/` — 3 CI workflows
- `release_odoo_module/` — Docker Compose for Odoo integration
- `app.py` — Streamlit dashboard entrypoint

---

## 2. Orchestration & Controlled Runtime

**Entrypoints (two tiers):**

1. `run_pipeline.py` (repo root) — MVP pipeline: extract → normalize → build dims/facts → write CSVs to `out/`. Invoked via `make run` or `python run_pipeline.py`.

2. `orchestration/run_pipeline.py` — Production orchestrator with the following deterministic execution controls:
   - `--pipeline` and `--env` flags select the logical pipeline and environment (`local | ci | prod`).
   - `--dry-run` flag exercises the event stream without any database writes.
   - `cmd_health()` — health check subcommand: queries `observability/sql/04_health_last_run.sql`; returns exit 0 if healthy (0 rows), exit 2 if stale/unhealthy (≥1 rows). Staleness threshold is `--max-age-minutes` (configurable).
   - `cmd_run()` — run subcommand: creates a UUID `run_id` via `02_run_start.sql`; iterates the step registry; executes pre-run gates, post-step gates, and end-run gates; writes final status via `03_run_finish.sql`.

**Gate contract:** Every gate and DQ check follows a BLOCKER pattern — any BLOCKER failure raises `RuntimeError` and marks the run `failed`. End-run gates execute unconditionally (even after step failure) to preserve diagnostic invariants.

**Event stream:** `orchestration/events/writer.py` emits typed events (`RUN_STARTED`, `STEP_STARTED`, `STEP_FINISHED`, `STEP_FAILED`, `RUN_FINISHED`) per execution.

**Observability SQL contracts** (`observability/sql/`):
- `01_pipeline_run_log.sql` — schema/table definition for run audit log
- `02_run_start.sql` — inserts run record, returns `run_id` (UUID)
- `03_run_finish.sql` — updates run record with status, rows_processed, error_message
- `04_health_last_run.sql` — staleness health check query

---

## 3. Dimensional Modeling

**Star schema confirmed.**

SQL dimension definitions found in `sql/analytics/`:
- `dim_customer.sql` — customer dimension with surrogate key (`customer_sk`), natural key (`customer_nk`), and SCD Type 2 temporal columns (`valid_from`, `valid_to`, `is_current`)
- `dim_date.sql` — date dimension with `date_id` (DATE) and `date_key` (INT, YYYYMMDD format)

Python-layer dimensional models (in `src/phc_analytics/transformations/`):
- `dim_customer.py`, `dim_product.py`, `dim_date.py` — Gold-layer dimension builders
- `fact_orders_enrich.py`, `fact_order_lines_enrich.py` — fact table enrichment (date key derivation, foreign key resolution)
- `agg_sales_by_product.py` — pre-aggregated serving layer

Output files confirmed in `out/`: `dim_customer.csv`, `dim_product.csv`, `dim_date.csv`, `fact_orders.csv`, `fact_order_lines.csv`, `agg_sales_by_product.csv`.

**SCD Type 2 confirmed.**

`sql/analytics/data_quality/dim_customer/01_scd2_integrity.sql` contains four explicit SCD2 integrity checks on `analytics.dim_customer`:
- No duplicate versions per natural key at the same `valid_from`
- Exactly one `is_current = TRUE` row per `customer_nk`
- Consistency between `is_current` and `valid_to` (`current → valid_to IS NULL`; `non-current → valid_to IS NOT NULL`)
- No overlapping temporal intervals per natural key (interval overlap detection via self-join with `COALESCE(..., 'infinity'::timestamptz)`)

---

## 4. Data Quality & Integrity Gates

**SQL DQ checks** (`sql/analytics/data_quality/`):

Contract: every check query must return **0 rows** to pass. Any returned row is a data quality failure. Checks are read-only, re-runnable, and deterministic.

| File | Asset | Checks |
|------|-------|--------|
| `data_quality/dq_dim_customer_scd2_integrity.sql` | `analytics.dim_customer` | Duplicate versions; single current row per NK; `is_current`/`valid_to` consistency; temporal overlap detection |
| `data_quality/dq_fact_orders_fk_customer.sql` | `analytics.fact_orders` | FK integrity: every `customer_id` must resolve to a current `dim_customer.customer_nk`; NULL/empty `customer_id` treated as invalid |
| `data_quality/dq_dim_date_unique_not_null.sql` | `analytics.dim_date` | `date_id` NOT NULL and UNIQUE; `date_key` NOT NULL and UNIQUE |

**DQ runner script:** `scripts/run_dq_folder.sh`
- Accepts a folder path; runs all `*.sql` files in sorted (deterministic) order via `psql`.
- Uses `-v ON_ERROR_STOP=1` and `-qAt` mode; treats any non-empty output as a gate failure.
- Exits 1 with `DQ GATE: FAILED` on any failure; exits 0 with `DQ GATE: PASSED` on full pass.

**Execution phases:**
- Pre-run: `get_pre_run_dq_checks()` + pre-run gates execute before any step
- Post-step: `get_post_step_dq_checks(step_id)` + post-step gates execute after each step completes
- End-run: `get_end_run_dq_checks()` + end-run gates execute unconditionally after all steps

**pytest data quality tests** (`tests/`):

- `test_dq_outputs.py` — validates pipeline CSV outputs: file existence for all 6 expected outputs; `customer_key` and `product_key` uniqueness and non-null; FK integrity between `fact_order_lines` and `dim_customer`, `dim_product`, `dim_date`; metric constraints (`quantity > 0`, `line_total >= 0`)
- `test_quality_fact_documents.py` — validates `fact_documents` parquet output: required schema columns present; `doc_id`, `doc_date`, `client_id`, `total` not null; `total >= 0`; `doc_type` in accepted domain `["FATURA", "RECIBO", "GUIA"]`; `doc_id` uniqueness
- `test_pipeline_contract.py` — validates partitioned parquet output contract: `partition_fact=True` produces `fact_documents` partitioned by `year_month`; partition filter returns correct rows; `year_month` column present in read result

---

## 5. Reproducibility & CI

**Local setup:**
- `pyproject.toml` — project packaged as `phc-analytics` v0.1.0; `requires-python = ">=3.9"`; runtime dependencies: `pandas`, `streamlit`, `plotly`, `snowflake-connector-python`, `ipykernel`, `jupyter`. Dev dependencies: `pytest`, `ruff`.
- `uv.lock` — deterministic dependency lock file (uv package manager).
- `Makefile` — three targets: `make run` (python run_pipeline.py), `make test` (pytest -q), `make clean` (rm out/*.csv out/*.parquet).
- `pytest.ini` — defines `integration` marker for slow tests.

**Docker:**
- `release_odoo_module/docker-compose.odoo.yml` — Docker Compose configuration for Odoo integration environment.

**CI workflows** (`.github/workflows/`):

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| `data-quality.yml` | push to `main`, all PRs | Spins up ephemeral PostgreSQL 16; seeds via `sql/ci/seed_dq_db.sql`; runs DQ checks for `dim_customer`, `fact_orders`, `dim_date` via `run_dq_folder.sh`; blocks merge on failure |
| `azure-smoke.yml` | scheduled (03:17 UTC daily), manual dispatch | Spins up PostgreSQL 15; bootstraps schema; runs production orchestrator (`orchestration/run_pipeline.py run`); runs health check with 1440-minute threshold |
| `governance-contract.yml` | PRs to `main` | Spins up PostgreSQL 16; applies RBAC bootstrap (`sql/ci/bootstrap_ci.sql`); runs governance contract (`sql/ci/governance_contract.sql`) |

---

## 6. Observability / Validation Outputs

**Dashboard screenshots** (in `media/`):

- `media/faturacao_mensal.png` — monthly billing chart
- `media/margem_mensal.png` — monthly margin chart
- `media/top_clientes.png` — top customers chart

**Pipeline audit log:** `observability/sql/01_pipeline_run_log.sql` defines the run log schema. The `azure-smoke.yml` CI workflow bootstraps this table on every scheduled run and verifies health post-execution.
