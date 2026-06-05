# João Fonseca — Data Engineer: Executive Summary

**Positioning:** Senior Data Engineer / Data Platform Builder with evidence-backed delivery across GCP, Azure, dbt, Terraform, Python, and SQL. GCP event-driven data platforms, Azure Databricks lakehouse, dbt on Snowflake, Python orchestration, and API ingestion. SQL-first architecture, data quality as a pipeline output, IaC-managed infrastructure.

Available for B2B contracts and full-time remote roles — Data Engineering, Data Platform, Analytics Engineering. Remote-first EU/UK/international.

---

## Project Portfolio at a Glance

| Project | Stack | Key Signal |
|---------|-------|------------|
| **Real-Time Data Platform (RTDP)** *(strongest proof asset)* | GCP · Pub/Sub · Cloud Run · BigQuery · Dataflow · Terraform · Python · dbt | GCP event-driven data platform; LEVEL_2 audit 85/100 ATTACK; 384 tests pass; Terraform IaC matched; 12 GCP service areas confirmed |
| **Azure Lakehouse ETL Platform** | Azure Databricks · ADLS Gen2 · Unity Catalog · Delta Lake · ADF · Python | Full medallion lakehouse; SQL-first Gold layer; declarative DQ; model contracts; executed against live Databricks SQL Warehouse with committed artifacts |
| dbt + Snowflake Analytics Platform | dbt Core · Snowflake · SQL | Three-layer Kimball star schema; SCD Type 2 snapshots; full dbt Docs lineage (manifest.json, catalog.json, run_results.json) |
| PHC Analytics Platform | Python · PostgreSQL · GitHub Actions | Medallion pipeline; production orchestrator with step registry and health check; SQL DQ gates; three CI workflows |
| GitHub Events Data Pipeline | Python · S3 · Airflow · Athena · GitHub Actions | End-to-end API ingestion to Athena analytics; Airflow TaskFlow DAG; structured run logs; contract validation |

---

## What I Build

**GCP event-driven data platforms**
End-to-end GCP data platforms using Pub/Sub ingestion, Cloud Run processing, BigQuery analytics, Terraform IaC, and dbt transformation. Workload Identity, Secret Manager, and Cloud Monitoring wired in by default.

**Lakehouse and cloud platforms**
Full Bronze → Silver → Gold medallion architectures on Azure Databricks and Unity Catalog. SQL-driven Gold layers. Declarative DQ layers with queryable views. Formal model contracts declaring grain and object type for every serving asset.

**Data modeling**
Kimball star schemas with dimension and fact tables. SCD Type 2 on both Snowflake (dbt snapshots) and PostgreSQL (SQL DDL). Date spines, zero-filled daily facts, referential integrity enforced at pipeline runtime.

**Python pipelines and orchestration**
Hardened API ingestion clients. Airflow TaskFlow DAGs. Production orchestrators with step registries, gate registries, and health check subcommands. Run logs written on every execution — success and failure paths.

**Data quality**
SQL zero-row contract gates. dbt generic tests (`not_null`, `unique`, `relationships`, `accepted_values`). pytest suites enforcing schema contracts, FK integrity, partition correctness. DQ results committed as artifacts, not just logged.

**CI and reproducibility**
GitHub Actions workflows blocking merges on quality gate failures. Locked dependencies (uv.lock, pyproject.toml). Makefile-driven local execution. Dockerised integration environments.

---

## Why Valuable for B2B Contracts

- **Evidence-backed delivery.** Every project has committed execution artifacts: run logs, DQ outputs, compiled manifests, schema catalogs. Claims are verifiable, not asserted.
- **GCP data platform depth.** RTDP demonstrates a full-stack GCP data platform: event ingestion, streaming proof, batch processing, analytics layer, and IaC — all in one evidence-backed proof asset (LEVEL_2 audit: 85/100 ATTACK).
- **SQL-first discipline.** The serving layer stays declarative. Python handles ingestion and orchestration; SQL handles modeling and serving.
- **Operational awareness.** Environment parameterization, idempotent execution, run modes, structured logging, and CI discipline are built in from the start — not bolted on.
- **Azure Databricks alignment.** The Azure lakehouse project directly targets the Azure/Databricks contract stack: Unity Catalog, Delta Lake, ADLS Gen2, ADF orchestration, Databricks SQL Warehouse.
- **Honest scoping.** Known gaps and limitations are documented per project. No inflated claims.

---

## How to Review This Pack

| Document | Purpose |
|----------|---------|
| `README.md` | Landing page — RTDP proof asset, GCP evidence, non-claims, full positioning |
| `EXECUTIVE_SUMMARY.md` | Full portfolio narrative and B2B positioning |
| `AZURE_LAKEHOUSE_FLAGSHIP.md` | Azure/Databricks flagship — most technically complete |
| `DBT_SNOWFLAKE_EVIDENCE.md` | dbt + Snowflake dimensional modeling evidence |
| `GITHUB_EVENTS_EVIDENCE.md` | API ingestion, Airflow, AWS/Athena stack evidence |
| `PLATFORM_SNIPPETS.md` | LinkedIn, Braintrust, ARC, Malt profile copy |
| `artefacts/azure_lakehouse/` | Committed execution evidence: run log, DQ status, Gold table sample |
