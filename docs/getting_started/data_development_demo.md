# Data Development Project Manual

This manual shows how to initialize a Datus data-development project from a
business requirement and reference SQL, then develop, review, execute, and
validate the requested mart. The sample package uses the `product_adoption`
dataset, which models Pendo feature usage events for account-level product
adoption analysis.

Use this page as an operating procedure: download the package, keep its project
structure intact, start the local database, initialize project knowledge, create
an implementation plan from the requirement, and complete the development
workflow with the bundled Datus skills.

## Project Inputs

The package contains two project inputs that drive the workflow:

| Input | Purpose |
|---|---|
| `docs/pendo_product_adoption_summary_requirements.md` | The business requirement for the target product adoption summary mart. Use it as the source of truth for scope, grain, fields, metrics, segmentation rules, and acceptance criteria. |
| `ref_sql/` | Historical SQL references. Use them to initialize project knowledge, extract lineage and reusable rules, and ground implementation decisions. |

The source data is Pendo feature interaction data:

| Source | Purpose |
|---|---|
| `raw.feature_event` | Feature usage events, including visitor, account, application, feature, event count, minutes, and timestamp. |
| `raw.feature_history` | Feature metadata used by reference SQL and project knowledge initialization. |
| `raw.page_history` | Page metadata used by reference SQL and feature enrichment examples. |

The target table to develop is:

```text
marts.pendo__product_adoption_summary
```

The target grain is:

```text
feature_id + account_id + app_id
```

The expected-result table is loaded with the sample data:

```text
marts.pendo__product_adoption_summary_expected
```

The project is complete when `marts.pendo__product_adoption_summary` reconciles
with the expected-result table.

## Package Layout

Download the package: [product_adoption.zip](../assets/product_adoption.zip).

After extraction, keep this directory structure intact:

```text
product_adoption/
  README.md
  docker-compose.yml
  pendo_start.duckdb
  docs/
    pendo_product_adoption_summary_requirements.md
  docker/
    duckdb-loader/
      Dockerfile
      requirements.txt
      load_duckdb_to_postgres.py
  ref_sql/
    staging/
      stg_pendo__feature_event.sql
      stg_pendo__feature_history.sql
      stg_pendo__page_history.sql
    intermediate/
      int_pendo__latest_feature.sql
      int_pendo__latest_page.sql
      int_pendo__feature_info.sql
      int_pendo__feature_daily_metrics.sql
    marts/
      feature.sql
      feature_event.sql
      feature_daily_metrics.sql
  .datus/
    skills/
```

## Workflow Overview

| Step | Operation | Result |
|---|---|---|
| 1 | Start PostgreSQL | Local database is available. |
| 2 | Load DuckDB data | `raw` source tables and `marts` expected-result table are copied into PostgreSQL. |
| 3 | Start Datus | Datus is ready with a configured model and datasource. |
| 4 | Initialize project knowledge | `project-set-up` generates reusable project documentation from `ref_sql/`. |
| 5 | Create the implementation plan | `etl-plan` turns the requirement and project context into an approved plan. |
| 6 | Implement after approval | Datus generates SQL jobs only after the plan is approved. |
| 7 | Review generated SQL | `sql-review` checks the implementation against the plan and requirement. |
| 8 | Execute jobs | `execute-job` creates the staging table and target mart. |
| 9 | Validate results | `data-compare` reconciles the target mart with the expected-result table. |

## Step 1: Start PostgreSQL

From the extracted project directory, start PostgreSQL:

```bash
cd product_adoption
docker compose up -d postgres
```

PostgreSQL connection values:

| Setting | Value |
|---|---|
| Host | `127.0.0.1` |
| Port | `5432` |
| Database | `pendo` |
| Username | `pendo` |
| Password | `pendo` |
| Default schema | `raw` |

## Step 2: Load Data

Run the one-time DuckDB-to-PostgreSQL migration:

```bash
docker compose --profile migration run --rm duckdb-loader
```

The loader copies these DuckDB schemas into PostgreSQL:

```text
raw
marts
```

Confirm that this expected-result table is available after migration:

```text
marts.pendo__product_adoption_summary_expected
```

Expected row count:

```text
24995
```

## Step 3: Start Datus and Configure the Datasource

Start Datus from the project directory:

```bash
datus
```

After Datus opens, configure the model in the Datus interface. Then configure
the datasource with these values:

| Setting | Value |
|---|---|
| Datasource name | `pendo_pg` |
| Type | `PostgreSQL` |
| Host | `127.0.0.1` |
| Port | `5432` |
| Database | `pendo` |
| Username | `pendo` |
| Password | `pendo` |
| Default schema | `raw` |

## Step 4: Initialize Project Knowledge

Use the `project-set-up` skill to reverse-initialize reusable project knowledge
from `ref_sql/`.

Enter this prompt in Datus:

```text
Initialize this project using skill project-set-up
```

Expected output documents:

| Document | Purpose |
|---|---|
| `AGENTS.md` | Project overview, architecture, core asset index, and key decisions. |
| `docs/business_knowledge.md` | Business rules, metric definitions, filters, special handling, and reusable business logic. |
| `docs/technical_standards.md` | SQL conventions for full reloads, schema bootstrap, timestamp parsing, naming, CTEs, window deduplication, and NULL handling. |
| `docs/table_lineage.md` | DAG and field lineage across retained staging, intermediate, and mart reference SQL. |
| `docs/ref_sql_inventory.md` | Per-file purpose, source tables, target tables, and SQL evidence. |

The initialization should analyze these reference SQL layers:

| Layer | Files |
|---|---|
| Staging | `stg_pendo__feature_event`, `stg_pendo__feature_history`, `stg_pendo__page_history` |
| Intermediate | `int_pendo__latest_feature`, `int_pendo__latest_page`, `int_pendo__feature_info`, `int_pendo__feature_daily_metrics` |
| Marts | `feature`, `feature_event`, `feature_daily_metrics` |

Before continuing, skim the generated docs. They are the project knowledge base
that later planning and implementation steps should use.

## Step 5: Create the Implementation Plan

Use the `etl-plan` skill to create a plan from the requirement document and the
initialized project knowledge.

Enter this prompt in Datus:

```text
Please create an ETL plan using skill etl-plan
```

The plan should define:

| Area | Expected content |
|---|---|
| Requirement boundary | Build `marts.pendo__product_adoption_summary`; do not build out-of-scope analytics tables. |
| Source and target objects | Source tables, staging table, target mart, and expected-result table. |
| Grain and metrics | `feature_id + account_id + app_id`, required output fields, adoption level rules, and feature health score logic. |
| Implementation jobs | SQL files to create under `jobs/`. |
| Validation approach | Row count, column comparison, numeric tolerance, and bidirectional difference checks. |

Expected plan file:

```text
plans/build_product_adoption_summary.md
```

Review the plan before allowing implementation. If the plan misses a requirement
from `docs/pendo_product_adoption_summary_requirements.md`, ask Datus to revise
the plan first.

Approve implementation with:

```text
Approve, start implementation the plan
```

## Step 6: Review the Generated SQL

After approval, Datus should generate SQL jobs such as:

```text
jobs/stg_pendo__feature_event.sql
jobs/pendo__product_adoption_summary.sql
```

Use the `sql-review` skill before execution:

```text
Please review the ETL SQL using skill sql-review
```

Review focus:

| Area | What to check |
|---|---|
| Requirement coverage | Output fields, target grain, adoption level rules, and feature health score match the requirement. |
| Source usage | SQL uses the intended source and staging tables. |
| PostgreSQL compatibility | DuckDB-style reference patterns are adapted correctly. |
| NULL and divide-by-zero handling | Ratio and score logic handles missing denominators explicitly. |
| Type consistency | Numeric outputs, especially `feature_health_score`, keep the expected type. |

If the review finds issues, update the SQL before execution.

## Step 7: Execute the SQL Jobs

After the review passes, execute the jobs with the `execute-job` skill:

```text
Please execute the SQL jobs using skill execute-job
```

Expected generated tables:

```text
staging.stg_pendo__feature_event
marts.pendo__product_adoption_summary
```

## Step 8: Validate the Result

Compare the generated mart with the expected-result table using the
`data-compare` skill:

```text
Please compare the job result with the expected table using skill data-compare
```

Validation should compare:

```text
marts.pendo__product_adoption_summary
marts.pendo__product_adoption_summary_expected
```

Acceptance criteria:

- 24,995 rows match.
- All 13 output columns match.
- Numeric comparison passes at sub-1e-9 tolerance.
- Bidirectional `EXCEPT` checks return no differences.
- No further SQL correction is required.

## Daily Startup

After the environment has already been initialized, start PostgreSQL and Datus
with:

```bash
cd product_adoption
docker compose up -d postgres
datus
```

The DuckDB file is only needed for initialization or a full rebuild.

## Skill Reference

The project includes Datus skills under:

```text
.datus/skills/
```

| Skill | Use |
|---|---|
| `project-set-up` | Initialize reusable project knowledge from SQL, docs, lineage, and business rules. |
| `etl-plan` | Create and confirm an implementation plan before generating SQL. |
| `sql-review` | Review generated ETL SQL against the approved plan and requirement. |
| `execute-job` | Execute SQL jobs and DDL-oriented table/job operations. |
| `data-compare` | Compare generated results against expected data and explain any differences. |
