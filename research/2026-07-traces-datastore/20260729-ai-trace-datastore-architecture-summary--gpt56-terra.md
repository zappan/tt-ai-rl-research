# AI Trace Datastore Architecture: Decision Summary

## Executive recommendation

AI training and evaluation traces should not be forced into one datastore. They combine two different workloads:

1. **Operational investigation:** engineers and users need to open a trace, inspect its span tree, follow tool calls, and filter recent failures quickly.
2. **Historical analysis:** teams need to compare models, prompts, datasets, evaluators, costs, and latency across a growing archive of runs.

The best general solution is a **hybrid architecture**:

```text
PostgreSQL                 Transactional control-plane data
ClickHouse                 Recent and interactive trace analytics
Object storage + Parquet   Durable raw payloads and low-cost long-term data
Apache Iceberg             Reliable analytical tables over Parquet, when needed
```

Use **PostgreSQL** for users, projects, experiment definitions, permissions, job state, and other data that requires transactions. Use **ClickHouse** when the product needs fast trace exploration and high-volume analytical queries. Keep large artifacts and the durable trace history in **object storage**. Store analytical data there as **Parquet** and add **Apache Iceberg** once that collection needs to behave like a shared production table rather than a set of files.

This design separates responsibilities cleanly:

- ClickHouse serves the interactive product experience.
- Iceberg and Parquet provide a durable, portable, economical system of record.
- PostgreSQL manages application state.
- Object storage holds payloads that are too large or too rarely accessed to place in an analytical database.

The result is better than choosing ClickHouse, Parquet, Iceberg, or PostgreSQL as the only answer. Each one solves an important part of the problem, but none solves all of it alone.

## The problem in clear terms

An AI trace records how an evaluation or agent run produced an outcome. It may contain the original input, model calls, tool calls, retrieved documents, parent-child spans, timings, token counts, costs, evaluator outputs, human annotations, errors, screenshots, recordings, and raw outputs.

At first, a trace looks like one nested JSON document. In practice, it changes while it is being produced. A run can begin before all spans arrive. A grader may add a score later. A human may add a label days later. The schema also changes as new models, tools, and evaluators are introduced.

The platform must therefore support all of the following:

| Need | What it means for storage |
|---|---|
| Fast trace lookup | Load a trace and its child spans without scanning a large archive. |
| Flexible analytics | Filter and group by high-cardinality values such as model version, prompt version, tool, evaluator, dataset, customer, and run. |
| Long retention | Preserve raw traces and artifacts for regression analysis, replay, audit, and future model improvement. |
| Evolving data | Accept optional fields, late scores, corrections, and new trace conventions. |
| Reliable ingestion | Let multiple writers publish data safely and give readers a consistent view. |
| Low operating cost | Avoid retaining every large payload in an always-on database. |
| Open access | Permit future use by multiple query engines without rewriting the archive. |

These needs conflict. Object storage is cheap and durable but is not fast at trace-by-ID lookup. A running analytical database is fast but costs more to retain indefinitely. Nested JSON is convenient for an API but inefficient when every analysis must inspect individual spans. A production trace system needs an architecture that acknowledges those tradeoffs.

## Recommended architecture

```text
SDKs, training workers, evaluation runners
                  |
                  v
       Trace ingestion and buffering layer
                  |
       +----------+-----------+----------------+
       |                      |                |
       v                      v                v
PostgreSQL              ClickHouse       Object storage
control plane           hot traces        raw JSON, artifacts,
                         and analytics    Parquet data files
                                               |
                                               v
                                        Apache Iceberg tables
                                               |
                                  DuckDB, Trino, Spark, Athena,
                                  or another approved query engine
```

The ingestion layer should use a stable span/event model, ideally aligned with OpenTelemetry concepts: `trace_id`, `span_id`, `parent_span_id`, timestamps, status, attributes, and links to artifacts. It should write data asynchronously and in batches where possible. A queue or buffer becomes important once individual events arrive faster than files or database parts should be created.

### PostgreSQL: application control plane

PostgreSQL is the right place for data where transactions, referential integrity, and small updates matter more than broad analytical scans. Typical tables include users, organizations, projects, experiments, dataset definitions, prompt definitions, permissions, task configuration, workflow state, and job records.

It can also be the first trace store for a small product. A compact early design can use typed columns for stable attributes and `jsonb` for uncommon metadata. This is appropriate when trace volume is modest and the primary need is to move quickly with one familiar database.

PostgreSQL stops being the best trace analytics engine when queries repeatedly scan very large time ranges or calculate percentile and grouped metrics across millions of spans. At that point, the workload is analytical rather than transactional.

### ClickHouse: interactive trace and span analytics

ClickHouse is the strongest default for an observability-style trace UI. It is suited to append-heavy data, high-cardinality filtering, time-range scans, aggregations, and concurrent analytical queries. It is particularly useful when users expect recent traces to appear quickly and dashboards to remain responsive.

Put frequently queried values in typed columns: timestamps, trace and span identifiers, parent identifiers, model and evaluator versions, dataset and experiment identifiers, span type, tool name, duration, tokens, cost, outcome, and error class. Store uncommon or evolving attributes in a JSON-like or map field rather than making the whole record opaque.

Do not routinely duplicate multi-megabyte prompts, recordings, screenshots, or full transcripts in every ClickHouse row. Store their object URI and content hash instead. The UI can fetch the original artifact when a user opens the trace.

ClickHouse is not automatically the durable source of truth. Keeping all raw history in its native storage can become unnecessarily expensive, and its operational role does not replace an archive, catalog, or table format.

### Object storage and Parquet: durable, low-cost history

Object storage is the right home for complete prompts and responses, raw OpenTelemetry exports, screenshots, recordings, datasets, checkpoints, tool transcripts, and older analytical data. It provides inexpensive retention at a scale that does not require an always-on database cluster.

Parquet is the preferred analytical file format because it is columnar, compressed, open, and supports nested values. Query engines can scan only needed columns rather than the entire trace payload. It works well for periodic model comparisons and large historical evaluation jobs.

Parquet alone, however, is a file format. It does not decide which files make up the current table, coordinate concurrent writers, make a multi-file commit atomic, manage schema changes, compact small files, or retain reproducible snapshots. A directory full of Parquet files is not yet a production table.

### Apache Iceberg: table management for the long-term store

Apache Iceberg supplies the table-management layer over Parquet files in object storage. It records table metadata and snapshots, so readers see a consistent set of data files. It also supports schema evolution, partition evolution, controlled updates and deletes, time travel, and metadata-based query planning.

Iceberg is recommended when at least several of these conditions are true:

- traces must be retained indefinitely and data volume is growing toward multiple terabytes;
- multiple writers or services publish data;
- schemas change as new trace dimensions are introduced;
- late data, corrections, deletions, or reprocessing are expected;
- different query engines need to use the same data;
- reproducibility requires querying a past snapshot;
- file compaction and partition changes must be managed safely.

Iceberg is not a database server. A working deployment still needs object storage, an Iceberg catalog, at least one query or compute engine, ingestion writers, and maintenance jobs. A simple production baseline is object storage, a REST or managed catalog, one query engine, and scheduled maintenance. Avoid introducing several engines, a streaming platform, and multiple catalogs before a real workload requires them.

## Data model: preserve the trace, analyze its spans

Nested JSON should be treated as an API and replay representation, not necessarily as the only physical layout. The most useful logical model has separate datasets:

| Dataset | Grain | Purpose |
|---|---|---|
| `traces` | One row per trace | Trace-level identifiers, timestamps, run metadata, outcome, and references to raw data. |
| `spans` | One row per span or event | Model calls, tool calls, graders, timing, errors, tokens, costs, and parent-child links. |
| `evaluation_results` | One row per evaluator result | Scores, pass/fail state, evaluator version, and explanations. |
| `artifacts` | One row per stored object | URI, content hash, media type, size, and retention metadata. |

The `spans` dataset should include at least `trace_id`, `span_id`, `parent_span_id`, `started_at`, `duration_ms`, `span_type`, status, and the dimensions that users frequently filter or group by. The application can reconstruct the trace tree from `trace_id` and `parent_span_id`.

This layout makes questions such as these inexpensive and clear:

- Which tool causes the most failures for a model version?
- What is the p95 model-call latency by dataset and prompt version?
- Which evaluator version changed the pass rate?
- Which traces contain three or more failed tool calls?

For exact replay, retain the original nested trace JSON as an immutable object or a complete trace record in the lake. This preserves fidelity without making every analytics query parse deep JSON arrays. A whole-trace row can be useful for exports and replay; a span row is better for ingestion, aggregation, and operational analytics. Use both representations when the product needs both jobs.

## Decision rules

### Start with PostgreSQL when

- the product is early, trace volume is limited, and one database is the best operational choice;
- transactions and frequent small updates dominate;
- analytical queries are limited to small time ranges;
- the team does not yet need a dedicated trace UI or long-term lakehouse.

Use a stable event schema from the start so append-only trace events can later be replicated into ClickHouse or written into Parquet without redesigning the product model.

### Use ClickHouse as the main active trace store when

- the product needs near-real-time trace visibility;
- people browse and filter traces interactively;
- dashboards and alerts execute frequently;
- high-cardinality analytics across recent data are common;
- trace-by-ID lookup needs low latency.

In this case, retain raw data and archive history in object storage at the same time. ClickHouse is the serving layer, not the only retention strategy.

### Use Iceberg and Parquet as the primary analytical record when

- long-term storage cost and open data access are central requirements;
- most analysis is batch-oriented and can tolerate seconds or longer;
- the data is read by several engines or teams;
- trace schemas and partitions will evolve;
- snapshots, corrections, and reproducibility matter.

An Iceberg-first design still benefits from a separate serving store or index when the user experience requires immediate trace navigation.

### Use plain Parquet directories only when

- one controlled writer owns ingestion;
- data is append-only and partitions are effectively immutable;
- schemas change rarely;
- queries are infrequent and mostly offline;
- manual compaction is acceptable.

This is a reasonable prototype or archive design. It is not a good long-term substitute for Iceberg once concurrent writers, corrections, evolution, and consistent snapshots become requirements.

## Why the common alternatives are incomplete

Several technologies can be useful additions, but they should not replace the architecture above without a specific reason.

| Alternative | Appropriate use | Why it is not the complete default |
|---|---|---|
| OpenSearch or Elasticsearch | Full-text search of prompts, outputs, and error text | It is more expensive for broad numerical analytics and long retention. |
| BigQuery or Snowflake | A managed warehouse for periodic organization-wide analysis | It can simplify operations, but does not by itself provide responsive trace navigation or artifact storage. |
| DynamoDB or Cassandra | Predictable key-based access at very high scale | They are poor fits for flexible multidimensional aggregations and percentile analysis. |
| Vector database | Finding semantically similar failures or outputs | It stores a derived similarity index, not the authoritative trace record. |

The key question is access pattern, not brand preference. A product that mostly runs overnight evaluations may use a managed warehouse instead of Trino or Spark. A product that requires rich prompt search may add OpenSearch. Those choices fit around the durable trace model; they do not change the need to separate transactional state, interactive analytics, raw artifacts, and long-term table management.

## Operating the lakehouse correctly

The main operational risk is treating object storage as a database without the required discipline. A reliable implementation must plan for:

- **Batching and file size:** do not write one Parquet file per trace. Buffer events into sufficiently large files and compact small files regularly.
- **Late data:** scores, annotations, and corrections should be written as append-only facts where possible, or handled with Iceberg updates when a current-state view is required.
- **Catalog availability:** the catalog is part of the data plane; back it up, secure it, and monitor its health.
- **Maintenance:** schedule compaction, manifest rewriting where appropriate, snapshot expiration, and orphan-file cleanup. These are normal duties, not exceptional repair tasks.
- **Schema governance:** assign stable field identities, evolve schemas deliberately, and keep writer and reader versions compatible.
- **Partitions and sort order:** partition for broad time and tenant/project pruning, then sort or cluster for common filters such as `trace_id`, model, evaluator, or run. Do not encode every possible query dimension in the physical path.
- **Security and retention:** protect raw prompts and outputs, enforce access controls in both the operational systems and query layer, and define deletion workflows for sensitive data.

## Phased adoption path

### Phase 1: prove the product and schema

Use PostgreSQL for control data and modest trace volumes, plus object storage for artifacts and raw payloads. Write trace events with stable identifiers and explicit schema versions. If an archive is needed, create partitioned Parquet files in batches rather than per event.

### Phase 2: establish a durable analytical record

Introduce an Iceberg catalog and register the Parquet-backed `traces`, `spans`, `evaluation_results`, and `artifacts` tables. Select one query engine that matches the team and cloud environment. Add scheduled compaction and metadata cleanup before table growth makes them urgent.

### Phase 3: add a serving layer where product latency requires it

Replicate recent or frequently queried spans into ClickHouse. Keep a defined hot-data window, such as the latest 30 to 90 days, based on observed access patterns rather than a fixed assumption. The full history remains in Iceberg; ClickHouse delivers fast UI and dashboard queries.

### Phase 4: expand only for demonstrated workloads

Add streaming ingestion, additional query engines, semantic search indexes, or advanced governance only when they solve a measured need. A vector database is a secondary similarity index, not the source of truth for traces. A text-search engine is optional for prompt/output search, not a replacement for analytical storage.

## Final decision

For a serious AI evaluation or observability platform, make **object storage plus Parquet and Iceberg** the durable analytical foundation, use **PostgreSQL** for transactional application state, and add **ClickHouse** for recent, interactive trace analysis. This preserves every trace economically, keeps the data portable, supports reproducible historical analysis, and still gives users a responsive trace experience.

Choose a simpler PostgreSQL-plus-object-storage design while the product is small. Choose Iceberg from the start if concurrent writers, evolving schemas, corrections, and a shared multi-engine archive are already known requirements. Add ClickHouse as soon as product behavior demonstrates that interactive trace navigation cannot tolerate lake query latency.

## Source material

This summary synthesizes:

- [AI Trace Storage, Parquet, and Apache Iceberg Architecture](20260729-ai-trace-storage-parquet-and-iceberg-architecture--gpt56-sol.md)
- [AI Trace Storage and Apache Iceberg Research Notes](20260729-ai-traces-iceberg-research--gpt55.md)
- [AI Trace Datastores and Apache Iceberg](20260729-ai-trace-datastore-apache-iceberg-research--gpt56-terra.md)
