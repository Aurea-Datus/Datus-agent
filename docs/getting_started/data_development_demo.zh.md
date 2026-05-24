# 数据开发项目操作手册

本手册说明如何基于业务需求和 Reference SQL 初始化一个 Datus 数据开发项目，
并完成开发、审查、执行和结果校验。示例包使用 `product_adoption` 数据集，
用 Pendo 功能使用事件模拟账号级产品采用度分析场景。

请把本文当作操作流程使用：下载示例包，保持项目目录结构不变，启动本地数据库，
初始化项目知识，基于需求创建实施计划，然后使用包内 Datus skills 完成开发流程。

## 项目输入

示例包包含两个驱动工作流的项目输入：

| 输入 | 作用 |
|---|---|
| `docs/pendo_product_adoption_summary_requirements.md` | 目标产品采用度 summary mart 的业务需求。它是范围、粒度、字段、指标、分层规则和验收标准的来源。 |
| `ref_sql/` | 历史 SQL 参考。用于初始化项目知识、抽取血缘和可复用规则，并为实现决策提供依据。 |

主要源数据是 Pendo 功能交互数据：

| Source | 作用 |
|---|---|
| `raw.feature_event` | 功能使用事件，包括 visitor、account、application、feature、事件次数、使用分钟数和时间戳。 |
| `raw.feature_history` | 功能元数据，用于 Reference SQL 和项目知识初始化。 |
| `raw.page_history` | 页面元数据，用于 Reference SQL 和功能信息增强示例。 |

需要开发的目标表是：

```text
marts.pendo__product_adoption_summary
```

目标表粒度是：

```text
feature_id + account_id + app_id
```

示例数据中已经包含 expected-result 表：

```text
marts.pendo__product_adoption_summary_expected
```

当 `marts.pendo__product_adoption_summary` 与 expected-result 表对账通过时，
项目完成。

## 示例包目录

下载包：[product_adoption.zip](../assets/product_adoption.zip)。

解压后保持目录结构不变：

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

## 工作流概览

| 步骤 | 操作 | 结果 |
|---|---|---|
| 1 | 启动 PostgreSQL | 本地数据库可用。 |
| 2 | 加载 DuckDB 数据 | `raw` 源表和 `marts` expected-result 表复制到 PostgreSQL。 |
| 3 | 启动 Datus | Datus 使用已配置的模型和 datasource 就绪。 |
| 4 | 初始化项目知识 | `project-set-up` 从 `ref_sql/` 生成可复用项目文档。 |
| 5 | 创建实施计划 | `etl-plan` 将需求和项目上下文转化为可审批计划。 |
| 6 | 审批后实现 | 计划审批后，Datus 生成 SQL jobs。 |
| 7 | 审查生成 SQL | `sql-review` 检查实现是否符合计划和需求。 |
| 8 | 执行 jobs | `execute-job` 创建 staging 表和目标 mart。 |
| 9 | 校验结果 | `data-compare` 将目标 mart 与 expected-result 表对账。 |

## 步骤 1：启动 PostgreSQL

在解压后的项目目录中启动 PostgreSQL：

```bash
cd product_adoption
docker compose up -d postgres
```

PostgreSQL 连接信息：

| 设置 | 值 |
|---|---|
| Host | `127.0.0.1` |
| Port | `5432` |
| Database | `pendo` |
| Username | `pendo` |
| Password | `pendo` |
| Default schema | `raw` |

## 步骤 2：加载数据

执行一次性 DuckDB 到 PostgreSQL 的迁移：

```bash
docker compose --profile migration run --rm duckdb-loader
```

loader 会把下面两个 DuckDB schema 复制到 PostgreSQL：

```text
raw
marts
```

迁移后确认 expected-result 表可用：

```text
marts.pendo__product_adoption_summary_expected
```

期望行数：

```text
24995
```

## 步骤 3：启动 Datus 并配置 datasource

在项目目录下启动 Datus：

```bash
datus
```

Datus 打开后，先在界面中配置模型。然后使用下面的值配置 datasource：

| 设置 | 值 |
|---|---|
| Datasource name | `pendo_pg` |
| Type | `PostgreSQL` |
| Host | `127.0.0.1` |
| Port | `5432` |
| Database | `pendo` |
| Username | `pendo` |
| Password | `pendo` |
| Default schema | `raw` |

## 步骤 4：初始化项目知识

使用 `project-set-up` skill 从 `ref_sql/` 反向初始化可复用项目知识。

在 Datus 中输入：

```text
Initialize this project using skill project-set-up
```

预期输出文档：

| 文档 | 作用 |
|---|---|
| `AGENTS.md` | 项目概览、架构、核心资产索引和关键决策。 |
| `docs/business_knowledge.md` | 业务规则、指标定义、过滤条件、特殊处理和可复用业务逻辑。 |
| `docs/technical_standards.md` | full reload、schema bootstrap、时间戳解析、命名、CTE、窗口去重和 NULL 处理等 SQL 约定。 |
| `docs/table_lineage.md` | retained staging、intermediate 和 mart Reference SQL 的 DAG 与字段血缘。 |
| `docs/ref_sql_inventory.md` | 每个文件的用途、源表、目标表和 SQL 证据。 |

初始化时应分析这些 Reference SQL 层级：

| 层级 | 文件 |
|---|---|
| Staging | `stg_pendo__feature_event`, `stg_pendo__feature_history`, `stg_pendo__page_history` |
| Intermediate | `int_pendo__latest_feature`, `int_pendo__latest_page`, `int_pendo__feature_info`, `int_pendo__feature_daily_metrics` |
| Marts | `feature`, `feature_event`, `feature_daily_metrics` |

继续之前，先快速浏览生成的文档。它们是后续计划和实现步骤要使用的项目知识库。

## 步骤 5：创建实施计划

使用 `etl-plan` skill 基于需求文档和已初始化的项目知识创建计划。

在 Datus 中输入：

```text
Please create an ETL plan using skill etl-plan
```

计划应定义：

| 范围 | 预期内容 |
|---|---|
| 需求边界 | 构建 `marts.pendo__product_adoption_summary`；不构建范围外 analytics 表。 |
| 源表和目标表 | 源表、staging 表、目标 mart 和 expected-result 表。 |
| 粒度和指标 | `feature_id + account_id + app_id`、必需输出字段、adoption level 规则和 feature health score 逻辑。 |
| 实现 jobs | 需要在 `jobs/` 下创建的 SQL 文件。 |
| 校验方式 | 行数、字段对比、数值 tolerance 和双向差异检查。 |

预期计划文件：

```text
plans/build_product_adoption_summary.md
```

审批实现前先审查计划。如果计划遗漏
`docs/pendo_product_adoption_summary_requirements.md` 中的需求，先要求 Datus
修订计划。

用下面的输入批准实施：

```text
Approve, start implementation the plan
```

## 步骤 6：审查生成的 SQL

审批后，Datus 应生成类似下面的 SQL jobs：

```text
jobs/stg_pendo__feature_event.sql
jobs/pendo__product_adoption_summary.sql
```

执行前使用 `sql-review` skill：

```text
Please review the ETL SQL using skill sql-review
```

审查重点：

| 范围 | 检查内容 |
|---|---|
| 需求覆盖 | 输出字段、目标粒度、adoption level 规则和 feature health score 是否符合需求。 |
| 源表使用 | SQL 是否使用预期源表和 staging 表。 |
| PostgreSQL 兼容性 | DuckDB 风格的参考 SQL 模式是否已正确适配。 |
| NULL 和除零处理 | ratio 和 score 逻辑是否显式处理缺失分母。 |
| 类型一致性 | 数值输出，尤其是 `feature_health_score`，是否保持预期类型。 |

如果审查发现问题，先修正 SQL 再执行。

## 步骤 7：执行 SQL jobs

审查通过后，使用 `execute-job` skill 执行 jobs：

```text
Please execute the SQL jobs using skill execute-job
```

预期生成的表：

```text
staging.stg_pendo__feature_event
marts.pendo__product_adoption_summary
```

## 步骤 8：校验结果

使用 `data-compare` skill 将生成的 mart 与 expected-result 表对比：

```text
Please compare the job result with the expected table using skill data-compare
```

校验对象：

```text
marts.pendo__product_adoption_summary
marts.pendo__product_adoption_summary_expected
```

验收标准：

- 24,995 行匹配。
- 13 个输出字段全部匹配。
- 数值比较在 sub-1e-9 tolerance 下通过。
- 双向 `EXCEPT` 检查没有差异。
- 不需要继续修正 SQL。

## 日常启动

环境初始化完成后，可以用下面的命令启动 PostgreSQL 和 Datus：

```bash
cd product_adoption
docker compose up -d postgres
datus
```

DuckDB 文件只在初始化或完整重建时需要。

## Skill 参考

项目包含的 Datus skills 位于：

```text
.datus/skills/
```

| Skill | 用途 |
|---|---|
| `project-set-up` | 从 SQL、文档、血缘和业务规则初始化可复用项目知识。 |
| `etl-plan` | 在生成 SQL 前创建并确认实施计划。 |
| `sql-review` | 基于已批准的计划和需求审查生成的 ETL SQL。 |
| `execute-job` | 执行 SQL jobs，以及偏 DDL 的 table/job 操作。 |
| `data-compare` | 将生成结果与 expected 数据对比，并解释差异。 |
