# DBT + Snowflake Analytics Platform — Technical Evidence

*Pack context: supporting project. Flagship project is Azure Lakehouse ETL Platform — see `AZURE_LAKEHOUSE_FLAGSHIP.md`.*

---

## 1. Architecture Overview

The project implements a three-layer dbt architecture targeting Snowflake:

- **Staging layer** (`models/staging/`): Raw source models that clean and type-cast input data. Sources include seed tables (`stg_customers`, `stg_orders`) and a demo model (`stg_my_first_model`). All staging models are tagged `staging` via `dbt_project.yml`.

- **Intermediate layer** (`models/intermediate/`): Business logic transformation models (`int_my_first_model`, `int_customers`) that sit between staging and marts. Tagged `intermediate`.

- **Marts layer** (`models/marts/`): Final consumption-ready models including:
  - `dim_customers` — customer dimension (active customers only)
  - `dim_customers_scd2` — SCD Type 2 customer dimension built from snapshot
  - `dim_date` — calendar date dimension generated via date spine
  - `fct_orders` — order-grain fact table
  - `fct_orders_daily` — daily aggregated orders fact
  - `fct_orders_daily_by_date` — date-complete daily fact (zero-filled via `dim_date` join)
  - `fct_my_first_model` — demo fact table

---

## 2. Dimensional Modeling

**Star schema confirmed.**

The mart layer contains clearly separated dimension tables (`dim_customers`, `dim_date`) and fact tables (`fct_orders`, `fct_orders_daily`, `fct_orders_daily_by_date`). Referential integrity is enforced via dbt `relationships` tests between fact and dimension keys.

**SCD Type 2 confirmed.**

`snapshots/customers_snapshot.sql` implements SCD Type 2 using the dbt `snapshot` materialisation:

- Strategy: `check`
- Monitored columns: `name`, `country`, `status`
- Unique key: `customer_id`
- Target schema: `SNAPSHOTS`

The snapshot produces `dbt_valid_from` and `dbt_valid_to` columns on the `customers_snapshot` table. The mart model `dim_customers_scd2` is built directly from this snapshot, exposing the full version history. Current rows are identified by `dbt_valid_to IS NULL`.

---

## 3. Data Quality Enforcement

The following dbt generic tests were detected across `models/staging/schema.yml`, `models/intermediate/schema.yml`, and `models/marts/schema.yml`:

**`not_null`**
Applied to primary keys and non-nullable columns across all layers:
- `stg_customers.customer_id`, `stg_orders.order_id`, `stg_orders.customer_id`, `stg_orders.order_date`, `stg_orders.amount`, `stg_orders.status`
- `int_my_first_model.id`
- `dim_customers.customer_id`, `dim_customers.country_code`
- `fct_orders.order_id`, `fct_orders.customer_id`, `fct_orders.order_date`, `fct_orders.amount`, `fct_orders.status`
- `dim_date.date_day`, `dim_date.year`, `dim_date.month`, `dim_date.day`, `dim_date.day_of_week_iso`
- `fct_orders_daily.*` (all metric columns)
- `fct_orders_daily_by_date.*` (all metric columns)
- `dim_customers_scd2.customer_id`, `.name`, `.country`, `.status`, `.dbt_valid_from`

**`unique`**
Applied to all primary keys:
- `stg_customers.customer_id`, `stg_orders.order_id`
- `dim_customers.customer_id`, `fct_orders.order_id`
- `dim_date.date_day`
- `fct_orders_daily.order_date`, `fct_orders_daily_by_date.order_date`

**`relationships`**
Referential integrity enforced between:
- `fct_orders.customer_id` → `dim_customers.customer_id`
- `fct_orders_daily.order_date` → `dim_date.date_day`
- `fct_orders_daily_by_date.order_date` → `dim_date.date_day`

**`accepted_values`**
Domain constraints applied to:
- `stg_customers.status`: `["active", "inactive"]`
- `stg_orders.status`: `["paid", "cancelled", "refunded"]`
- `fct_orders.status`: `["paid", "cancelled", "refunded"]`
- `dim_customers_scd2.country`: `["PT", "FR", "ES", "IT", "UK", "DE"]`
- `dim_customers_scd2.status`: `["active", "inactive"]`

---

## 4. Observability & Documentation

- **dbt Docs**: `target/index.html` is present. dbt documentation site was generated for this project.
- **Lineage**: `target/manifest.json` is present. Full DAG lineage (node graph) is available via the dbt docs interface and programmatically via the manifest.
- **Run results**: `target/run_results.json` is present. Execution metadata including model run status, timing, and test outcomes is captured.
- **Source freshness**: Configured on the `demo_de.orders` source table via `loaded_at_field: ORDER_DATE`, with warn threshold at 24 hours and error threshold at 48 hours.

---

## 5. Reproducibility

- **`dbt_project.yml` detected**: Project name `de_snowflake_dbt`, version `1.0.0`, profile `de_snowflake_dbt`. Layer tagging (`staging`, `intermediate`, `marts`) and seed column type overrides are declared.
- **Execution artefacts present**:
  - `target/manifest.json` — compiled project graph
  - `target/catalog.json` — Snowflake schema catalogue
  - `target/run_results.json` — last execution results
  - `target/index.html` — generated documentation site
