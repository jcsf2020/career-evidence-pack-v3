# João Fonseca — Career Evidence Pack v3

**Positioning:** Senior Data Engineer / Data Platform Builder with evidence-backed delivery across GCP, Azure, dbt, Terraform, Python, and SQL.

Available for B2B contracts and full-time remote roles — Data Engineering, Data Platform, Analytics Engineering. Remote-first EU/UK/international.

---

## Project Portfolio

| Project | Stack | Key Signal |
|---------|-------|------------|
| **Real-Time Data Platform (RTDP)** *(strongest proof asset)* | GCP · Pub/Sub · Cloud Run · BigQuery · Dataflow · Terraform · Python · dbt | GCP event-driven data platform; LEVEL_2 audit 85/100 ATTACK; 384 tests pass; Terraform IaC matched; 12 GCP service areas confirmed |
| **Azure Lakehouse ETL Platform** | Azure Databricks · ADLS Gen2 · Unity Catalog · Delta Lake · ADF · Python | Full medallion lakehouse; SQL-first Gold layer; declarative DQ; model contracts; executed against live Databricks SQL Warehouse |
| **dbt + Snowflake Analytics Platform** | dbt Core · Snowflake · SQL | Three-layer Kimball star schema; SCD Type 2 snapshots; full dbt Docs lineage |
| **GitHub Events Data Pipeline** | Python · S3 · Airflow · Athena · GitHub Actions | End-to-end API ingestion to Athena analytics; Airflow TaskFlow DAG; structured run logs |

---

## RTDP — GCP Event-Driven Data Platform

**Strongest public proof asset. LEVEL_2 audit: 85/100 ATTACK.**

Repository: `jcsf2020/real-time-data-platform`

This is a portfolio and proof asset, not a claimed customer production system.

### Architecture

GCP-native event-driven data platform using Pub/Sub ingestion, Cloud Run processing, BigQuery analytics, and Terraform infrastructure-as-code.

### Confirmed Evidence

- Terraform IaC state matches deployed GCP infrastructure (PLAN_EXIT=0, no changes).
- 384 automated tests pass.
- Read-only GCP CLI evidence captured 2026-06-01 confirmed 12 GCP service areas supporting this proof asset.

### GCP Services Confirmed (2026-06-01 CLI evidence)

| Service | Role |
|---------|------|
| Pub/Sub | Event ingestion; push subscription to Cloud Run; DLQ and retry policy |
| Cloud Run | Push-subscription worker and Cloud Run Jobs (BigQuery append, dbt refresh, silver refresh) |
| BigQuery | Analytical layer with defined schemas, partitioning, clustering, Terraform-managed metadata |
| Cloud SQL | Relational storage layer |
| Dataflow | Apache Beam streaming proof (europe-west1; two historical jobs recorded) |
| Cloud Scheduler | Job orchestration |
| Cloud Build | CI / build pipeline |
| Artifact Registry | Container image registry |
| Secret Manager | Secrets management |
| Cloud Monitoring | Observability |
| Logging | Structured logging |
| IAM / Workload Identity | Service account and identity management |

### Safe Summary

Built and deployed a GCP-based event-driven data platform using Pub/Sub, Cloud Run, BigQuery, Cloud SQL, Dataflow, Cloud Scheduler, Cloud Build, Artifact Registry, Secret Manager, Logging, and Monitoring. Terraform IaC state matches deployed infrastructure (PLAN_EXIT=0). 384 automated tests pass. LEVEL_2 audit score: 85/100 ATTACK.

---

## What This Evidence Does Not Claim

| Non-claim | Reason |
|---|---|
| Sustained production throughput | The documented load test is bounded evidence, not a long-running production SLO. |
| Exactly-once delivery semantics | The platform is documented with realistic distributed-system constraints. |
| Live customer production usage | RTDP is a portfolio/proof asset, not presented as a customer production system. |

---

## Positioning

**Senior Data Engineer / Data Platform Builder**

- GCP: Pub/Sub · Cloud Run · BigQuery · Dataflow · Terraform · Workload Identity · Monitoring
- Azure: Databricks · ADLS Gen2 · Unity Catalog · Delta Lake · ADF
- Analytics: dbt Core · Snowflake · SQL-first serving
- Language: Python · SQL
- Delivery: Evidence-backed, IaC-managed, test-covered, CI-integrated
- Available: Remote-first EU/UK/international — B2B contracts and full-time roles

---

## How to Review This Pack

| Document | Purpose |
|----------|---------|
| `EXECUTIVE_SUMMARY.md` | Full portfolio narrative and B2B positioning |
| `AZURE_LAKEHOUSE_FLAGSHIP.md` | Azure/Databricks flagship evidence — most technically complete |
| `DBT_SNOWFLAKE_EVIDENCE.md` | dbt + Snowflake dimensional modeling evidence |
| `GITHUB_EVENTS_EVIDENCE.md` | API ingestion, Airflow, AWS/Athena stack evidence |
| `PLATFORM_SNIPPETS.md` | LinkedIn, Braintrust, ARC, Malt profile copy |
| `artefacts/azure_lakehouse/` | Committed execution evidence: run log, DQ status, Gold table sample |
