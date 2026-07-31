# AI Trace Storage: Problem, Options, and Recommended Architecture

**Date:** 2026-07-29  
**Audience:** Technical decision-makers

## Executive answer

AI trace storage is not one storage problem. It combines several workloads that pull the architecture in different directions:

- an application must quickly load a trace and reconstruct its spans;
- analysts must compare models, prompts, datasets, and evaluators across large histories;
- the system must accept spans, scores, labels, and corrections that arrive late;
- raw prompts, responses, media, and other artifacts may need to be retained indefinitely;
- schemas evolve as models, tools, evaluators, and trace conventions change.

No single datastore handles all of these needs equally well. The strongest general solution is a layered architecture:

```text
Trace and evaluation producers
            │
            ▼
     Buffered ingestion
            │
     ┌──────┼───────────────┐
     ▼      ▼               ▼
PostgreSQL  ClickHouse      Object storage
control     optional hot    durable artifacts and
plane       serving layer   Iceberg/Parquet history
                                │
                                ▼
                     Batch and ad-hoc query engines
```

Use **PostgreSQL** for transactional application state. Use **object storage with Parquet** for low-cost, durable data. Add **Apache Iceberg** when those Parquet files need reliable table semantics. Add **ClickHouse** when users need near-real-time, interactive trace exploration.

The key design principle is to make object storage the durable home of the complete history while treating low-latency databases as serving layers for the data that benefits from them.

## 1. The problem in practical terms

An AI trace records how a model or agent produced a result. It may contain prompts, responses, parent-child spans, tool calls, retrieved documents, timings, token usage, evaluator scores, human labels, screenshots, recordings, and other artifacts.

Although an API or user interface often presents this as one nested JSON document, the underlying system must support two very different access patterns.

### Interactive investigation

A user may want to:

- open one trace immediately after it finishes;
- inspect its inputs, outputs, and errors;
- navigate parent and child spans;
- filter recent spans by model, tool, score, or customer;
- refresh dashboards and latency percentiles frequently.

This workload favors a continuously running database with fast point lookups, indexes or sorting, and predictable low latency.

### Historical analysis

An evaluation or research job may want to:

- compare many model and prompt versions over months of data;
- aggregate scores by dataset and evaluator version;
- recalculate cost, token, and latency distributions;
- reproduce results as they appeared at an earlier point in time;
- process millions or billions of spans in one scan.

This workload favors inexpensive columnar storage and query engines that can read only the required columns across a large history.

These needs should not be forced into one storage system merely to make the architecture appear simpler. Doing so usually makes either interactive performance too slow or long-term retention unnecessarily expensive.

## 2. What each storage option is good at

| Option | Best use | Main strength | Main limitation |
|---|---|---|---|
| PostgreSQL | Control-plane state and smaller trace workloads | Transactions, updates, and operational simplicity | Large analytical scans eventually become expensive |
| ClickHouse | Hot, interactive trace and span analytics | Fast filtering and aggregation over high-cardinality events | Requires an always-on analytical database and is not the cheapest permanent archive |
| Plain Parquet on object storage | Small or controlled batch-oriented datasets | Low cost, portability, compression, and broad engine support | Files alone provide no table transactions, snapshots, or coordinated maintenance |
| Iceberg with Parquet | Shared, evolving analytical history | Reliable table metadata, snapshots, schema evolution, and multi-engine access | Requires a catalog, query engine, maintenance, and operational ownership |
| Search or vector database | Full-text or semantic discovery | Specialized retrieval | Should be a secondary index, not the trace system of record |

### PostgreSQL

PostgreSQL is a good starting point when the system is small, trace data is closely connected to application entities, and updates or transactions matter more than large analytical scans. Important fields should use typed columns, while less predictable metadata can use `jsonb`.

Its long-term role should remain clear even after the analytical system grows: users, projects, experiments, permissions, configurations, workflow state, and task definitions belong in a transactional control plane.

### ClickHouse

ClickHouse is the strongest default when the product itself is an interactive trace explorer. It fits append-heavy event data, high-cardinality filters, and repeated aggregations over spans. It is appropriate when traces must appear quickly, dashboards refresh frequently, and many users query recent data concurrently.

ClickHouse does not have to be the permanent home of every raw payload. Recent or frequently queried structured data can be loaded into it while the complete history remains in object storage.

### Parquet on object storage

Parquet is an efficient, open columnar file format. It compresses analytical data well, supports nested structures, and can be read by many systems. Object storage makes it economical to retain complete historical traces and large artifacts for long periods.

Parquet is not a database or table manager. A directory of Parquet files does not by itself provide atomic multi-file commits, consistent snapshots, coordinated schema changes, row-level corrections, or automatic compaction. It also performs poorly if ingestion creates a very large number of tiny files.

Plain Parquet is reasonable when one writer controls an append-only dataset, schemas rarely change, queries are mostly offline, and occasional manual maintenance is acceptable.

### Apache Iceberg

Iceberg is a table-management layer over files such as Parquet. It tracks which files belong to a table and which snapshot is current. It also supports schema and partition evolution, consistent commits, time travel, and managed updates or deletes.

Iceberg becomes valuable when any combination of these conditions is present:

- multiple services or engines read and write the data;
- schemas or partition layouts evolve;
- spans, labels, or scores arrive late;
- records must be corrected, reprocessed, or deleted;
- reproducible historical snapshots matter;
- compaction and metadata management must be systematic;
- object storage is intended to be the durable source of truth.

Iceberg is not a database server. A working deployment still needs object storage, an Iceberg catalog, a query or compute engine, ingestion, and scheduled maintenance.

## 3. Recommended data model

Do not make the analytical model one opaque JSON document per trace. That representation is useful for APIs, replay, and user interfaces, but it is awkward for span-level filtering and aggregation.

Use a small set of related datasets instead:

```text
traces
  one row per trace or evaluation run

spans
  one row per model call, tool call, evaluator step, or other span

evaluation_results
  one row per evaluator result, score, or label

artifacts
  one row per external prompt, response, image, recording, or large payload
```

Link them with stable identifiers such as `trace_id`, `span_id`, `parent_span_id`, `run_id`, and `evaluation_id`. Store frequently filtered dimensions—timestamps, model and evaluator versions, dataset identifiers, status, token counts, latency, cost, and error type—as typed columns. Keep uncommon or provider-specific attributes in a flexible map or JSON field.

Also retain the original trace payload or export as an immutable artifact when exact replay, audit, or future reprocessing matters. This gives the system both an efficient analytical representation and a faithful source record.

## 4. Best target architecture

### Durable system of record

Use **Iceberg tables backed by Parquet in object storage** for the complete analytical history when the production requirements already justify a table format. Store large binary or document artifacts directly in object storage and reference them by URI and content hash.

This layer provides low-cost retention, open access from multiple engines, and reproducible snapshots without tying the durable data to one database server.

### Transactional control plane

Use **PostgreSQL** for mutable application state: projects, users, experiments, access control, configurations, workflows, and the status needed to coordinate ingestion or evaluation jobs.

PostgreSQL can also hold traces during an early prototype, but the architecture should avoid making large historical analytical scans a permanent responsibility of the control-plane database.

### Query engines

Use the smallest engine that satisfies the workload:

- **DuckDB** for development, local exploration, and smaller datasets;
- **Trino, Athena, Spark, or an existing warehouse** for distributed historical queries and maintenance;
- **ClickHouse** for recent or frequently queried data when an interactive product experience requires it.

The durable format remains the same as query needs evolve. New engines can be introduced without rewriting the historical source of truth.

### Optional hot serving layer

Add ClickHouse when the system needs near-real-time visibility, fast trace-ID lookup, frequent dashboards, arbitrary filtering across millions of spans, or high query concurrency.

A common hot/cold policy is:

```text
Recent or frequently queried traces  → ClickHouse
Complete retained history            → Iceberg and Parquet
Large raw artifacts                  → Object storage
Transactional application state      → PostgreSQL
```

The exact hot-data retention period should be based on observed product usage and cost, not fixed prematurely.

## 5. Adoption path

### Stage 1: keep the first version small

For a prototype or modest single-writer workload, use PostgreSQL plus object storage, or disciplined partitioned Parquet with explicit schemas and buffered file creation. Avoid one file per trace.

### Stage 2: introduce Iceberg when table management is needed

Add an Iceberg catalog and scheduled maintenance once the dataset becomes shared, mutable, multi-engine, or operationally difficult to manage as files.

Start with object storage, one catalog, one query engine, and scheduled maintenance. Add more engines only when a concrete workload requires them.

If concurrent writers, corrections, time travel, or multi-engine access are known requirements from the beginning, start with Iceberg instead of building a temporary loose-file convention.

### Stage 3: add interactive serving only when justified

Introduce ClickHouse when measured latency or concurrency requirements show that direct lake queries cannot provide the desired user experience. Populate it from the ingestion stream or from committed Iceberg data, and keep the lakehouse as the complete retained history.

## 6. Operational obligations

The lakehouse reduces storage cost, but it does not remove operations. Production ownership must include:

- buffering incoming events into appropriately sized files;
- compacting small files;
- rewriting bloated metadata or manifests when needed;
- expiring old snapshots under a defined retention policy;
- deleting orphaned files safely;
- monitoring catalog and object-storage health;
- testing compatibility among writers, readers, catalogs, and Iceberg versions;
- securing both metadata and data paths;
- backing up catalog state and testing recovery procedures.

These tasks are part of the architecture, not optional cleanup. A system that ignores them will accumulate slow query planning, excessive metadata, wasted storage, and recovery risk.

## 7. Decision guide

Choose **PostgreSQL plus object storage** when the product is early, volume is modest, transactions matter, and operational simplicity is the priority.

Choose **plain Parquet on object storage** when there is one writer, data is append-only, schemas are stable, queries are mostly offline, and the team accepts manual file management.

Choose **Iceberg with Parquet** when object storage will be the durable analytical system of record and the system needs consistent commits, evolution, corrections, snapshots, compaction, or multiple engines.

Choose **Iceberg plus ClickHouse** when the system needs both inexpensive indefinite retention and an interactive trace-analysis product with near-real-time data and high query concurrency.

## Final recommendation

For a production AI evaluation and trace platform, the best long-term design is:

```text
PostgreSQL                    transactional control-plane state
Iceberg + Parquet             canonical analytical trace history
Object storage                large artifacts and durable files
DuckDB or distributed SQL     batch and ad-hoc analysis
ClickHouse, when required     interactive hot trace serving
```

This architecture keeps the durable data inexpensive and portable, preserves fast application behavior where it matters, and lets each component solve the workload it was designed for. The important choice is not “Parquet or ClickHouse.” It is deciding which data must be durable, which must be interactive, and which system should own each responsibility.

## Source material

This synthesis is based exclusively on the following repository documents:

- `20260729-ai-trace-datastore-apache-iceberg-research--gpt56-terra.md`
- `20260729-ai-trace-storage-parquet-and-iceberg-architecture--gpt56-sol.md`
- `20260729-ai-traces-iceberg-research--gpt55.md`

The source documents contain the longer technical analysis, examples, operational details, and external references behind this summary. Their claims were consolidated here but not independently revalidated.
