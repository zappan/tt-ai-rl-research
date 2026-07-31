# AI Trace Datastore Summary

**Compiled:** 2026-07-29  
**Model:** GPT-5.5  
**Purpose:** Clear synthesis of the repository's AI trace datastore research

## Source material

This summary is based on the three AI trace datastore research files in this
repository:

- `20260729-ai-traces-iceberg-research--gpt55.md`
- `20260729-ai-trace-datastore-apache-iceberg-research--gpt56-terra.md`
- `20260729-ai-trace-storage-parquet-and-iceberg-architecture--gpt56-sol.md`

Two related derivative summaries present in the directory were also checked for
unique decision points:

- `20260729-ai-trace-datastore-architecture-summary--gpt56-terra.md`
- `20260729-ai-trace-storage-synthesis--gpt56-sol.md`

Those files assemble and expand two source conversations:

- [Best datastore for AI traces](https://chatgpt.com/share/6a69b3bf-68a0-83eb-9cef-6e4aa03c91d4)
- [Apache Iceberg Technical Details](https://chatgpt.com/share/6a69b3cd-fe24-83ed-bd41-4827100c9823)

No new external research was performed for this document. Links and factual
claims are carried forward from the existing repository material.

## Executive summary

AI training and evaluation traces are not just logs, JSON documents, or
application records. They combine several workloads:

- operational metadata about users, experiments, jobs, permissions, and
  configurations;
- high-volume span and event analytics;
- raw prompts, outputs, tool transcripts, screenshots, recordings, datasets,
  checkpoints, and other large artifacts;
- long-term historical data that may be reanalyzed months or years later;
- interactive trace debugging where a user expects one trace to load quickly.

No single datastore is the best fit for all of those jobs. The strongest
production design is a layered architecture:

```text
Evaluation and training workers
            |
            v
  OpenTelemetry-style ingestion
            |
     +------+----------------+----------------+
     |                       |                |
     v                       v                v
PostgreSQL              ClickHouse       Object storage
control plane           hot trace        raw payloads,
metadata                analytics        artifacts,
                                         Parquet files
                                             |
                                             v
                                      Apache Iceberg
                                      table metadata
                                      and commits
```

The best solution is usually:

- **PostgreSQL** for transactional control-plane data.
- **ClickHouse** for recent, searchable, interactive trace analytics.
- **Object storage** for raw payloads, artifacts, and durable low-cost history.
- **Parquet** as the columnar analytical file format.
- **Apache Iceberg** as the table layer over Parquet once the data becomes
  shared, large, concurrent, or production-critical.

For an early prototype, PostgreSQL plus object storage can be enough. For a
serious evaluation platform, design the trace model around spans and events
from the beginning, then add ClickHouse and Iceberg/Parquet as scale and
query needs justify them.

The most important conceptual point is this:

> Nested JSON is the API and UI shape of a trace. It does not have to be the
> physical storage shape.

A trace can be presented as one nested document while being stored as rows in
`traces`, `spans`, `scores`, and `artifacts` tables. That representation is
much easier to query, aggregate, retain, compact, and evolve.

## The problem

AI evaluation platforms need to store enough information to explain how a model
or agent produced an answer. A single evaluation trace may contain:

- the original prompt;
- model inputs and outputs;
- parent-child spans;
- tool calls and tool results;
- retrieval context;
- token counts and latency;
- costs;
- evaluator scores;
- reward values;
- pass/fail status;
- human labels;
- error types;
- screenshots, recordings, files, and other artifacts;
- model, dataset, prompt, evaluator, and environment versions.

That data often appears in products as nested JSON:

```json
{
  "trace_id": "trace-123",
  "model": "claude",
  "task": {
    "id": "task-77",
    "dataset": "browser-evals"
  },
  "steps": [
    {
      "id": "step-1",
      "type": "model",
      "input": {"prompt": "Find the price"},
      "output": {"text": "I'll search..."}
    },
    {
      "id": "step-2",
      "type": "tool",
      "tool": {"name": "browser"},
      "output": {"status": 200}
    }
  ],
  "grade": {
    "reward": 0.8,
    "passed": true
  }
}
```

That nested shape is useful for APIs and human inspection, but the underlying
storage problem is broader than document storage. The platform must support two
very different access patterns.

First, it needs **interactive investigation**. A user may want to load one
trace, inspect its span tree, open tool-call inputs and outputs, compare a
failed run to a previous run, and filter recent failures. These workflows need
low latency and convenient lookup by identifiers such as `trace_id`, `run_id`,
`example_id`, `model_version`, and `error_type`.

Second, it needs **historical analytics**. A team may want to compare models,
prompts, datasets, evaluators, tasks, token usage, costs, score distributions,
latency percentiles, and pass rates over millions or billions of spans. These
queries are analytical scans over high-cardinality dimensions.

Those two access patterns stress storage systems differently. A design that is
excellent for cheap long-term retention may be awkward for interactive trace
navigation. A design that is excellent for dashboards may be too expensive as
the permanent archive for every prompt, response, screenshot, and trace.

The problem is therefore not "which datastore is best?" in isolation. The real
question is:

> Which responsibilities should be separated, and which storage layer should
> serve each responsibility?

## Core requirements

An AI trace datastore should be evaluated against these requirements.

| Requirement | Why it matters |
|---|---|
| Fast trace lookup | Debugging tools need to load one trace and its spans quickly. |
| Span-level analytics | Evaluation analysis often groups by model, tool, dataset, evaluator, and error. |
| High cardinality | Fields such as run, prompt version, model version, dataset, and customer can have many values. |
| Long retention | Historical traces are useful for regression analysis, replay, audit, and model improvement. |
| Raw-data preservation | Analytical rows should not destroy the original prompt, output, transcript, or artifact. |
| Schema evolution | Trace structure changes as tools, providers, evaluators, and annotations change. |
| Late data | Scores, labels, spans, and enrichments may arrive after the trace starts. |
| Concurrent ingestion | Multiple workers may write traces at the same time. |
| Open access | Data should remain usable by multiple query engines and tools. |
| Operational sustainability | Databases, catalogs, compaction jobs, indexes, and query engines all need ownership. |

These requirements conflict. For example, object storage is cheap and durable,
but it does not naturally provide low-latency point lookup. ClickHouse is strong
for interactive analytics, but it is not the cheapest place to keep every large
artifact forever. PostgreSQL is simple and transactional, but it becomes a poor
fit for broad analytical scans over hundreds of millions of spans.

## Recommended mental model

Separate the trace platform into four kinds of data:

| Data type | Examples | Best home |
|---|---|---|
| Control-plane metadata | users, projects, permissions, experiments, jobs, configs | PostgreSQL |
| Hot analytical trace rows | spans, scores, token usage, latency, failures, recent runs | ClickHouse |
| Raw and large payloads | prompts, outputs, screenshots, recordings, datasets, checkpoints | Object storage |
| Durable analytical history | historical trace/span/scores tables | Iceberg tables backed by Parquet |

This separation avoids forcing unrelated workloads into one datastore. It also
lets each layer scale independently.

## Trace shape: nested JSON versus span rows

A trace is usually a tree or directed graph of spans. The important identifiers
are:

```text
trace_id
span_id
parent_span_id
```

The UI can reconstruct a nested trace by grouping spans by `trace_id` and
linking each span to its parent. That means the physical tables can look like
this:

```text
traces
  trace_id
  created_at
  project_id
  run_id
  dataset_id
  model_name
  model_version
  reward
  passed
  metadata

spans
  trace_id
  span_id
  parent_span_id
  sequence_number
  span_type
  started_at
  duration_ms
  model_name
  tool_name
  input_ref
  output_ref
  attributes

scores
  trace_id
  evaluator_name
  evaluator_version
  score
  passed
  explanation_ref

artifacts
  artifact_id
  trace_id
  span_id
  uri
  content_hash
  media_type
  size_bytes
```

This design keeps common query fields in typed columns while preserving flexible
metadata in JSON-like columns or maps.

Storing each trace as one opaque JSON document is acceptable early, but it
becomes limiting. It makes common analytical questions harder:

```text
Which tool produced the most failures?
What is p95 model-call latency by model version?
Which dataset version regressed after a prompt change?
Which evaluator version changed pass rates?
Which runs had more than three failed tool calls?
```

For those questions, one row per span or event is usually better. The original
nested JSON can still be stored as an immutable replay artifact in object
storage.

## PostgreSQL

PostgreSQL is the right default for transactional application state and often a
good first datastore for a prototype.

Use PostgreSQL for:

- users, teams, projects, and permissions;
- experiment definitions;
- model and dataset registry metadata;
- evaluator configuration;
- workflow and job state;
- task definitions;
- small-to-medium trace tables;
- annotations and operational records that need transactions.

A practical prototype can use:

```text
PostgreSQL + JSONB
S3-compatible object storage
```

PostgreSQL typed columns should hold fields that are frequently filtered or
joined. `jsonb` can hold flexible attributes while the schema is still evolving.
PostgreSQL also supports JSON indexing and native partitioning, which can help
while the system is small or medium-sized.

PostgreSQL becomes less attractive when the workload is dominated by large
analytical scans:

```text
Compare 50 model versions across 200 million examples,
grouped by dataset, evaluator, customer, prompt version,
latency percentile, and cost.
```

That is an OLAP workload. At that point, PostgreSQL should remain the
control-plane database, but traces should flow into an analytical store.

## ClickHouse

ClickHouse is the strongest default for high-volume interactive trace analytics.
AI traces have many properties that match ClickHouse well:

- append-heavy ingestion;
- timestamps and durations;
- millions or billions of events;
- high-cardinality fields;
- repeated aggregations;
- dashboards and filtering;
- numerical metrics such as pass rate, token usage, cost, and latency.

ClickHouse is particularly strong when the product itself is an observability or
trace exploration UI. Users expect recent traces to appear quickly, filters to
respond interactively, and latency/cost/error charts to update without waiting
for a batch lakehouse query.

Use ClickHouse for fields such as:

```text
trace_id
span_id
parent_span_id
timestamp
duration_ms
experiment_id
run_id
dataset_id
example_id
model_provider
model_name
model_version
checkpoint
temperature
seed
evaluator_name
evaluator_version
score
passed
error_type
input_tokens
output_tokens
cost
latency_ms
tags
attributes
artifact_urls
```

ClickHouse can also store nested or semi-structured data using types such as
`JSON`, `Map`, `Array`, `Tuple`, and `Nested`. The key is not to hide every
important field inside one opaque JSON blob. Promote common dimensions to typed
columns and keep unusual attributes in flexible columns.

The source research notes that W&B publicly documents ClickHouse for Weave
trace storage in its self-managed architecture, along with ClickHouse Keeper
and S3-compatible object storage. The broader W&B platform also uses MySQL for
transactional metadata, Redis for caching and queues, and object storage for
models, datasets, files, and artifacts. See the W&B self-managed documentation:

- [W&B Weave self-managed](https://docs.wandb.ai/weave/guides/platform/weave-self-managed?utm_source=chatgpt.com)
- [W&B self-managed hosting](https://docs.wandb.ai/platform/hosting/hosting-options/self-managed?utm_source=chatgpt.com)

The source material did not identify authoritative public documentation for
HUD.ai's hosted backend datastore. The safe conclusion is:

- W&B Weave: ClickHouse is publicly documented.
- HUD.ai: backend datastore not publicly confirmed from the repository's source
  material.

ClickHouse is not necessarily the permanent source of truth for all data. It is
often best as the hot, low-latency serving layer for recent or frequently
queried traces.

## Object storage

Object storage is the right home for large data and indefinite retention.

Store these in S3-compatible object storage or an equivalent system:

- complete prompts and responses;
- raw nested trace JSON;
- tool-call transcripts;
- retrieved documents;
- screenshots and recordings;
- multimodal inputs and outputs;
- datasets and training examples;
- model checkpoints;
- evaluator explanations;
- archived OpenTelemetry exports;
- Parquet analytical files.

Analytical stores should keep object URIs and content hashes rather than
duplicating large payloads in every row:

```text
s3://ai-evals/runs/2026/07/run-123/example-456/output.json
sha256:...
```

This keeps hot analytical tables smaller while preserving exact replay data.

## Parquet

Parquet is a physical file format for columnar analytical storage. It is a good
fit for AI traces because it is compressed, widely supported, and can represent
nested structures.

Parquet is especially useful for:

- low-cost long-term retention;
- batch evaluation;
- offline historical analysis;
- portability across engines;
- storing analytical data in object storage.

Engines and tools that can work with Parquet include DuckDB, ClickHouse, Spark,
Trino, Athena, BigQuery external tables, Snowflake external tables, PyArrow, and
Polars. The source material references the Apache Parquet documentation:

- [Apache Parquet](https://parquet.apache.org/?utm_source=chatgpt.com)
- [Parquet overview](https://parquet.apache.org/docs/overview/?utm_source=chatgpt.com)

However, Parquet is not a database. Plain Parquet files do not provide:

- table transactions;
- snapshots;
- atomic multi-file commits;
- row-level updates;
- schema coordination;
- automatic file compaction;
- concurrency control;
- a global table index;
- partition evolution;
- cleanup of replaced or orphaned files.

Plain Parquet directories are reasonable when:

- one writer controls the data;
- data is append-only;
- schemas rarely change;
- queries are mostly offline;
- the dataset is not enormous;
- manual compaction is acceptable.

For example:

```text
s3://traces/
  project_id=123/
    event_date=2026-07-28/
      hour=20/
        part-0001.parquet
        part-0002.parquet
```

This is a valid starting point, but it becomes fragile as the system grows.

The most common failure mode is the small-file problem. Traces arrive one at a
time, but Parquet works best when files are large enough for efficient
compression, metadata, and scans. Writing one Parquet file per trace is a bad
design:

```text
trace-1.parquet
trace-2.parquet
trace-3.parquet
...
trace-10000000.parquet
```

Instead, ingestion should buffer events and write larger files:

```text
Incoming spans
      |
      v
Kafka / queue / buffer
      |
      v
Batch every 30-300 seconds
      |
      v
128-512 MB Parquet file
```

The exact size depends on the engine and workload, but batching and compaction
are fundamental requirements.

## Apache Iceberg

Apache Iceberg is not a database server. It is an open table format plus client
libraries that define how analytical tables are represented, versioned,
committed, and discovered.

A production Iceberg setup usually includes:

```text
Query / processing engines
  Spark, Flink, Trino, DuckDB, ClickHouse, Athena, etc.
          |
          v
Iceberg catalog
  REST, AWS Glue, Hive Metastore, JDBC, Nessie, vendor catalog
          |
          v
Object storage
  S3, GCS, Azure Blob, MinIO, HDFS
          |
          v
Data and metadata files
  Parquet / ORC / Avro data
  Avro manifests
  JSON table metadata
```

Iceberg adds the table-management layer that loose Parquet files lack:

- snapshots;
- atomic commits;
- schema evolution;
- partition evolution;
- hidden partitioning;
- time travel;
- manifest metadata;
- delete files;
- copy-on-write and merge-on-read update patterns;
- consistent reads for concurrent writers;
- metadata-driven query planning.

An Iceberg table contains data files, delete files, manifest files, and table
metadata. A snapshot points to a manifest list; the manifest list points to
manifests; manifests point to data and delete files. Query engines use this
metadata tree to plan reads without listing every object in a bucket or opening
every Parquet footer.

Iceberg becomes valuable when:

- there are multiple writers;
- schemas evolve;
- data arrives late;
- files need compaction;
- queries need consistent snapshots;
- partitions need to change over time;
- records may be corrected or deleted;
- reproducibility matters.

For AI evaluation data, snapshots and time travel are especially useful:

```text
Reproduce the metrics exactly as they appeared
before the evaluator logic changed.
```

Iceberg is still infrastructure. It does not eliminate operational work. It
moves the work from one database into a lakehouse stack:

- ingestion buffering;
- Parquet writers;
- compaction;
- catalog service;
- metadata cleanup;
- orphan-file cleanup;
- query engine capacity;
- schema governance;
- monitoring failed commits;
- dependency and engine-version compatibility.

Production Iceberg should be treated as a managed data platform, not as a
folder of files.

## Implementation guardrails

The durable lakehouse is only reliable if it is operated deliberately. The
important guardrails are:

- **Batch incoming events.** Do not write one Parquet file per trace or span.
  Buffer writes and compact small files on a schedule.
- **Keep table metadata healthy.** Monitor catalog availability, commit
  failures, manifest growth, snapshot retention, and orphaned files.
- **Partition for broad pruning.** Use coarse dimensions such as time,
  project, tenant, or environment. Do not encode every possible filter into the
  object-store path.
- **Sort or cluster for common lookups.** Common filters such as `trace_id`,
  `run_id`, model, evaluator, or timestamp should influence sort order or
  clustering where the chosen engine supports it.
- **Treat late facts carefully.** Prefer append-only facts for scores,
  annotations, and corrections where possible. Use Iceberg updates or deletes
  when a current-state table is required.
- **Protect raw payloads.** Prompts, outputs, screenshots, and retrieved
  documents may contain sensitive data. Access controls, retention policy, and
  deletion workflows must cover both metadata and object storage.
- **Keep versions pinned.** Query engines, writers, Iceberg libraries, and
  catalogs must be tested together. Avoid casual "latest" dependencies in
  production data infrastructure.

## Comparing the main options

| Option | Best use | Main weakness |
|---|---|---|
| PostgreSQL | Prototype, control-plane metadata, transactional workflows | Eventually poor for huge analytical scans |
| ClickHouse | Hot trace analytics, interactive dashboards, high-cardinality aggregation | Another database cluster to operate; not cheapest archive |
| Plain Parquet | Simple append-only offline analytics | No table transactions, snapshots, concurrency, or compaction management |
| Iceberg + Parquet | Durable analytical system of record on object storage | Requires catalog, engines, compaction, and operational ownership |
| Elasticsearch/OpenSearch | Full-text search over prompts and outputs | Expensive storage; weaker for broad numerical analytics |
| BigQuery/Snowflake | Managed warehouse analytics | Query and ingestion costs; less natural for trace UI navigation |
| DynamoDB/Cassandra | Massive predictable key-value access | Poor fit for flexible multidimensional analytics |
| Vector database | Similarity search over failures or prompts | Secondary index only; not a primary trace datastore |

## Best approach by stage

### Prototype

Use this when the system is small, product requirements are still changing, and
operational simplicity matters most:

```text
PostgreSQL + JSONB
S3-compatible object storage
```

Keep a clean span-oriented data model even if it starts in PostgreSQL. Store
large payloads in object storage from the beginning. This avoids a painful
redesign later.

### Growing product

Use this when trace volume grows and users need analytical exploration:

```text
PostgreSQL
  control-plane metadata

ClickHouse
  recent traces, spans, scores, metrics, dashboards

Object storage
  raw traces, payloads, artifacts, archives
```

At this stage, ClickHouse becomes the main interactive trace store. Object
storage remains the durable home for large and raw payloads.

### Durable analytical platform

Use this when retention, portability, reproducibility, and long historical
analysis become first-class requirements:

```text
PostgreSQL
  control plane

ClickHouse
  hot interactive serving layer

Iceberg + Parquet on object storage
  canonical analytical history

Object storage
  raw payloads and artifacts
```

This is the strongest long-term architecture for a serious AI evaluation
platform. It avoids using ClickHouse as the only archive, and it avoids using
object storage as if it were a low-latency database.

## Decision rules

Choose **PostgreSQL only** when:

```text
the system is early
+ data volume is modest
+ transactions matter
+ analytics are simple
+ operational simplicity matters most
```

Choose **ClickHouse** when:

```text
users need interactive trace exploration
+ traces must appear quickly
+ many analytical filters and aggregations are common
+ high-cardinality dimensions matter
+ dashboards and operational views are product features
```

Choose **Iceberg + Parquet** when:

```text
long retention matters
+ object-storage cost matters
+ batch analytics are common
+ query latency of seconds is acceptable
+ multiple engines should read the same data
+ reproducible historical snapshots matter
```

Choose **ClickHouse plus Iceberg/Parquet** when:

```text
the product needs both:
  low-latency interactive trace exploration
  complete low-cost analytical history
```

That hybrid architecture is the most balanced solution.

## Recommended final architecture

For a production AI trace and evaluation platform, use this architecture:

```text
SDKs / evaluation runners / training jobs
              |
              v
OpenTelemetry-style events and ingestion API
              |
              v
Buffer / stream / batch writer
              |
     +--------+----------------+----------------+
     |                         |                |
     v                         v                v
PostgreSQL                ClickHouse       Object storage
control plane             hot analytical   raw payloads,
metadata                  trace serving    artifacts,
                                           replay data
                                                |
                                                v
                                      Iceberg + Parquet
                                      canonical analytical
                                      history
```

Use these table responsibilities:

```text
PostgreSQL
  projects
  users
  permissions
  experiments
  datasets
  evaluator configs
  workflow state

ClickHouse
  traces
  spans
  scores
  token usage
  latency
  costs
  errors
  hot artifact references

Iceberg + Parquet
  trace history
  span history
  score history
  artifact index history
  replay/audit snapshots

Object storage
  full raw JSON
  prompts and outputs
  screenshots and recordings
  retrieved documents
  datasets
  checkpoints
  evaluator explanations
```

Use optional secondary indexes only for specific access patterns:

- full-text index for searching prompt/output text;
- vector index for semantic similarity over failures or prompts;
- key-value lookup index for fast `trace_id -> location` retrieval if the
  lakehouse is used directly by a UI.

Those indexes should not become the source of truth.

## Best possible solution

The best possible solution is not a single datastore. It is a system that
separates control, serving, archive, and analysis:

1. Model traces as spans and events, not only as opaque nested JSON.
2. Keep important query dimensions in typed columns.
3. Store large raw payloads in object storage with content hashes and stable
   URIs.
4. Use PostgreSQL for transactional metadata and workflow state.
5. Use ClickHouse for hot, interactive trace exploration and analytical
   dashboards.
6. Use Iceberg tables backed by Parquet for complete durable analytical
   history.
7. Add specialized indexes only when a concrete search or lookup pattern needs
   them.

This design gives the platform:

- fast recent trace debugging;
- scalable historical analysis;
- low-cost retention;
- open data portability;
- exact replay of raw data;
- room for schema evolution;
- a migration path from prototype to production.

The simplest phrasing is:

> PostgreSQL owns the application state. ClickHouse serves the interactive
> trace product. Object storage keeps the heavy truth. Iceberg and Parquet make
> the long-term analytical history reliable and portable.

That is the clearest and strongest architecture described by the repository's
research material.
