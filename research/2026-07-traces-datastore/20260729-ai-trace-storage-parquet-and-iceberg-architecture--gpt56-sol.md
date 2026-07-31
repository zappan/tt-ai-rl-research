# AI Trace Storage, Parquet, and Apache Iceberg Architecture

**Compiled:** 2026-07-29  
**Research sources:** [Best datastore for AI traces](https://chatgpt.com/share/6a69b3bf-68a0-83eb-9cef-6e4aa03c91d4) and [Apache Iceberg Technical Details](https://chatgpt.com/share/6a69b3cd-fe24-83ed-bd41-4827100c9823)

## Problem statement

AI training and evaluation systems generate traces that must explain not only a final model response, but the complete path that produced it. A single evaluation may include prompts, model responses, parent-child spans, tool calls, retrieved documents, timings, token usage, evaluator scores, human labels, screenshots, recordings, and other artifacts. Although this is naturally presented by an API or user interface as nested JSON, the underlying workload combines several materially different storage and query problems.

Trace data is predominantly append-heavy, but it is not always immediately immutable. A trace may begin before all of its spans arrive; automated graders may run later; people may add labels; scores may be corrected; and metadata may be enriched after ingestion. The structure also evolves as models, tools, evaluators, and trace conventions change. Meanwhile, data volume can grow from a modest collection of evaluation runs into millions or billions of spans, with historical traces retained indefinitely because they may later be valuable for regression analysis, replay, audit, or model improvement.

The system therefore needs to serve two very different access patterns:

- **Interactive investigation:** load one trace quickly, reconstruct its hierarchy, inspect its inputs and outputs, navigate parent and child spans, and find related failures.
- **Historical analytics:** compare models, prompts, datasets, evaluators, customers, tools, latency percentiles, costs, and score distributions across large time ranges.

The architectural problem is to decide whether those needs should be served by one datastore or by a layered system—and, specifically, what roles should be assigned to PostgreSQL, ClickHouse, object storage, Parquet, and Apache Iceberg.

## Required capabilities and competing pressures

The storage design must be evaluated against the following requirements:

| Requirement | Why it matters |
|---|---|
| Fast trace lookup and navigation | Debugging and evaluation interfaces need responsive point lookups and span-tree reconstruction. |
| High-cardinality analytics | Queries commonly group and filter by model version, dataset, prompt, evaluator, tool, customer, run, and experiment. |
| Exact replay and raw-data preservation | Normalized analytical rows must not prevent recovery of the original prompt, response, transcript, or artifact. |
| Nested and evolving schemas | Trace structures change and contain optional, provider-specific, or deeply nested attributes. |
| Affordable long-term retention | The complete history may be too large or valuable to expire aggressively. |
| Late events and corrections | Scores, labels, spans, and enrichment may arrive after the initial trace is written. |
| Concurrent, reliable ingestion | Production writers need safe commits and predictable visibility without corrupting shared analytical state. |
| Open access and portability | The data should remain usable by multiple engines and avoid unnecessary storage-format lock-in. |
| Operational sustainability | Catalogs, databases, compaction, cleanup, caching, and query engines all impose real ownership costs. |

These pressures pull in different directions. An always-on analytical database favors low-latency exploration, while object storage favors inexpensive durability and effectively unconstrained capacity. Nested documents favor natural replay, while span-level rows favor aggregation. Loose Parquet files minimize initial infrastructure, while a table format adds the coordination and maintenance semantics required by concurrent production use.

The decision under examination is therefore not simply “which database is best?” It is:

> Which combination of transactional metadata storage, hot trace analytics, durable analytical files, and table-management infrastructure provides the right balance of query performance, retention cost, portability, and operational burden for this workload?

## Executive synthesis

There is no single datastore that naturally serves every part of an AI training and evaluation platform. The combined recommendation is a deliberately separated architecture:

```text
Evaluation and training workers
            │
            ▼
  OpenTelemetry / ingestion layer
            │
     ┌──────┼───────────────┐
     ▼      ▼               ▼
PostgreSQL  ClickHouse      Object storage
control     hot analytical  durable payloads,
plane       traces/spans    Parquet data files
                                │
                                ▼
                         Apache Iceberg
                    table metadata and commits
                                │
                 ┌──────────────┼─────────────┐
                 ▼              ▼             ▼
               DuckDB          Trino        Spark/Flink
```

The central decisions are:

- **Use ClickHouse when interactive trace exploration matters.** It is the strongest default for high-volume, append-heavy, high-cardinality trace analytics, especially when users expect low-latency filtering, aggregation, and span-by-span navigation.
- **Use PostgreSQL for transactional control-plane state.** Experiments, users, permissions, configurations, task definitions, workflow state, and early-stage trace workloads fit naturally there. Typed columns plus `jsonb` are sufficient until analytical scans become dominant.
- **Use object storage for indefinite retention and large payloads.** Raw traces, prompts, responses, screenshots, datasets, recordings, checkpoints, and cold analytical history belong in S3-compatible storage.
- **Use Parquet as the analytical file format, not as the entire table-management system.** Parquet supplies efficient columnar storage and nested types, but loose files do not supply atomic multi-file commits, snapshots, schema coordination, compaction, or concurrency control.
- **Add Iceberg once the Parquet collection becomes a shared production table.** Iceberg supplies metadata, snapshots, schema and partition evolution, transactional commits, and maintenance operations over immutable Parquet files. It is not a standalone database server and still requires a catalog, object store, query engine, and scheduled maintenance.
- **Treat nested JSON as an interchange and presentation model.** A trace UI can show one nested document while the analytical representation stores one row per span or event, linked by `trace_id`, `span_id`, and `parent_span_id`. Preserve the original JSON separately when exact replay is important.
- **A hybrid hot/cold design is often the economic optimum.** Keep recent or frequently queried traces in ClickHouse and the complete history in Iceberg/Parquet. A query engine can scan the lake for batch work, while a compact trace index or hot store supports direct trace lookup.
- **Operational complexity is moved, not eliminated.** Replacing ClickHouse with a lakehouse trades database-cluster operation for ingestion buffering, file sizing, compaction, catalog availability, metadata cleanup, query-engine capacity, and version compatibility management.

## Method and preservation note

This document is a thematic synthesis of the two linked ChatGPT conversations. It reorganizes their material by architectural concern rather than reproducing the conversations chronologically. All substantive recommendations, trade-offs, examples, schemas, operational cautions, sizing guidance, and adoption criteria from the four model answers are retained below. Repeated conclusions remain where they provide context for a different decision.

The shared pages expose the model's final answers, brief planning commentary, search-activity summaries, and elapsed-time reasoning recaps. They do **not** expose private chain-of-thought. The visible reasoning artifacts are preserved exactly in the next section; no hidden reasoning has been inferred or reconstructed. Claims and links are carried over from the source conversations and have been reformatted, not independently revalidated for this compilation.

## Visible research-process notes

| Topic | Visible artifact | Preserved text |
|---|---|---|
| Datastore selection | Planning commentary | I’ll frame this as an architecture choice, because “best” depends mainly on trace volume, query patterns, and retention. I’ll compare the strongest options and give a practical default. |
| Datastore selection | Research activity | Searched 24 websites |
| Datastore selection | Reasoning recap | Worked for 10s |
| HUD.ai and W&B storage | Research activity | Searched 2 websites |
| HUD.ai and W&B storage | Reasoning recap | Worked for 14s |
| Parquet versus ClickHouse | Planning commentary | The real trade-off is not “Parquet versus ClickHouse” so much as **cheap durable files versus an always-on query engine**. I’ll compare them specifically for trace ingestion, debugging, and evaluation analytics. |
| Parquet versus ClickHouse | Research activity | Searched 21 websites |
| Parquet versus ClickHouse | Reasoning recap | Worked for 10s |
| Apache Iceberg operations | Planning commentary | I’ll separate what Iceberg itself requires from what your query engine, catalog, and object store require, then give you a practical production baseline rather than a misleading single “minimum spec.” |
| Apache Iceberg operations | Research activity | Searched 18 websites |
| Apache Iceberg operations | Reasoning recap | Worked for 22s |

## 1. Workload architecture and datastore responsibilities

**Source question:** What is the best data store for storing AI models training evaluation traces data?

For **AI training and evaluation traces**, the best general-purpose datastore is usually **ClickHouse**, backed by **object storage** for large artifacts and long-term retention.

### Recommended architecture

```text
Training/evaluation workers
          │
          ▼
 OpenTelemetry / event collector
          │
          ├──► ClickHouse
          │      Structured traces, metrics, scores, metadata
          │
          └──► S3-compatible object storage
                 Prompts, outputs, datasets, images,
                 checkpoints, detailed payloads, Parquet archives
```

#### Why ClickHouse is the strongest default

Evaluation traces typically have these properties:

- append-heavy writes;
- millions or billions of events;
- timestamps and durations;
- highly variable metadata;
- high-cardinality fields such as model version, dataset, run, prompt, evaluator and experiment;
- analytical queries over large time ranges;
- aggregations such as pass rate, latency percentiles, token usage and score distributions.

That is fundamentally an **OLAP/event analytics workload**, which is where ClickHouse excels. Its own observability guidance specifically targets logs and traces, while its column-oriented layout allows analytical queries to scan only the relevant columns. ([ClickHouse](https://clickhouse.com/docs/use-cases/observability/introduction?utm_source=chatgpt.com))

OpenTelemetry is a good ingestion model because it represents traces as collections of spans connected through parent-child relationships. That maps naturally to evaluation runs, model calls, tool calls, graders and subtasks. ([OpenTelemetry](https://opentelemetry.io/docs/concepts/signals/traces/?utm_source=chatgpt.com))

### What should go where?

#### ClickHouse

Store fields you frequently filter, group or aggregate:

```text
trace_id
span_id
parent_span_id
timestamp
duration_ms

experiment_id
run_id
evaluation_id
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

Put common dimensions in typed columns. Keep unusual or evolving attributes in a map or JSON-like column, rather than making the entire record an opaque JSON document.

#### Object storage

Store large or infrequently queried objects:

- complete prompts and responses;
- multimodal input and output;
- tool-call transcripts;
- retrieved documents;
- model checkpoints;
- training examples;
- evaluator explanations;
- raw OpenTelemetry exports;
- archived traces in Parquet.

Parquet is column-oriented and designed for efficient compressed analytical storage, making it a good long-term archival format. ([Parquet](https://parquet.apache.org/?utm_source=chatgpt.com))

Store the object URI and content hash in ClickHouse:

```text
s3://ai-evals/runs/2026/07/run-123/example-456/output.json
sha256:...
```

Avoid putting multi-megabyte prompts, images or full agent transcripts directly into every ClickHouse row.

### When PostgreSQL is enough

Use **PostgreSQL first** when:

- the system is still relatively small;
- you expect perhaps thousands or low millions of evaluations rather than billions of spans;
- evaluation data is closely tied to application entities;
- transactions and frequent updates matter;
- operational simplicity matters more than maximal analytical throughput.

A practical initial design is:

```sql
evaluation_runs
evaluation_examples
trace_spans
scores
artifacts
```

Use ordinary typed columns for important attributes and `jsonb` for less predictable metadata. PostgreSQL supports GIN indexing for searching keys and values inside `jsonb`, and native table partitioning can help as trace volume grows. ([PostgreSQL](https://www.postgresql.org/docs/current/datatype-json.html?utm_source=chatgpt.com))

PostgreSQL becomes less attractive when you repeatedly run queries such as:

```sql
Compare 50 model versions across
200 million examples,
grouped by dataset, evaluator, customer,
prompt version and latency percentile.
```

That is the point where ClickHouse is usually the more natural engine.

### Other options

| Store | Best use | Main limitation |
|---|---|---|
| **ClickHouse** | High-volume trace analytics | Another specialized database to operate |
| **PostgreSQL** | Small-to-medium systems and metadata | Eventually expensive for huge analytical scans |
| **S3 + Parquet + Iceberg** | Cheap long-term storage and offline analysis | Less suitable for interactive trace exploration alone |
| **Elasticsearch/OpenSearch** | Full-text search through prompts and outputs | Higher storage cost; weaker for broad numerical analytics |
| **BigQuery/Snowflake** | Managed warehouse and periodic analysis | Ingestion/query costs and less responsive trace navigation |
| **Cassandra/DynamoDB** | Massive predictable key-based access | Poor fit for flexible multidimensional evaluation analysis |
| **Vector database** | Semantic search over prompts or failures | Not a primary trace datastore |

A vector database should be treated as a **secondary index**, not the source of truth. Store embeddings there only when you need queries such as “find failures semantically similar to this one.”

### My practical recommendation

For a new production platform:

```text
ClickHouse       primary analytical trace store
PostgreSQL       experiments, users, configs and workflow state
S3 + Parquet     payloads, datasets, artifacts and cold history
OpenTelemetry    trace/event interchange and collection
```

For an early prototype:

```text
PostgreSQL + JSONB
S3-compatible object storage
```

Design the event schema around OpenTelemetry concepts from the beginning. Once PostgreSQL analytics become uncomfortable, stream or replicate the append-only events into ClickHouse without redesigning the entire evaluation model.

**Bottom line:** use **ClickHouse for the traces, object storage for the heavy data, and PostgreSQL for control-plane metadata**. That combination provides better scaling and query ergonomics than trying to force all three workloads into one datastore.

## 2. Nested JSON traces and product architecture

**Source question:** We were looking at tools like hud.ai and wandb, and was wondering what data store they use for their traces, which apparently look like nested JSONs

Yes—what you see in HUD or W&B as a **nested JSON trace** is usually the **API/UI representation**, not necessarily how it is physically stored.

### W&B Weave

W&B publicly documents that **Weave uses ClickHouse for trace storage**. Its self-managed architecture includes:

- a ClickHouse cluster for traces;
- ClickHouse Keeper for coordination;
- S3-compatible object storage for persistence and larger objects. ([Weights & Biases Documentation](https://docs.wandb.ai/weave/guides/platform/weave-self-managed?utm_source=chatgpt.com))

So the earlier ClickHouse recommendation closely matches what W&B itself does.

The broader W&B platform also uses:

- **MySQL** for transactional platform metadata;
- **Redis** for caching and queues;
- **object storage** for models, datasets, files and artifacts. ([Weights & Biases Documentation](https://docs.wandb.ai/platform/hosting/hosting-options/self-managed?utm_source=chatgpt.com))

Conceptually:

```text
W&B Models metadata       → MySQL
Weave traces/evaluations  → ClickHouse
Models, datasets, files   → S3-compatible storage
Cache and queues          → Redis
```

### HUD.ai

HUD publicly documents that each rollout produces a replayable trace containing things such as prompts, tool calls and grades. ([hud](https://docs.hud.ai/building/environments-as-data?utm_source=chatgpt.com))

However, I could not find authoritative public documentation identifying HUD’s underlying production database. Their SDK and documentation expose the data model and cloud API, but not enough of the hosted backend to state confidently whether they use ClickHouse, PostgreSQL, DynamoDB or another system.

Therefore:

- **W&B Weave: confirmed ClickHouse**
- **HUD.ai: backend datastore not publicly confirmed, as far as I could verify**

### How nested JSON maps to ClickHouse

Suppose the UI returns this:

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
      "tool": {
        "name": "browser",
        "arguments": {"url": "https://example.com"}
      },
      "output": {"status": 200}
    }
  ],
  "grade": {
    "reward": 0.8,
    "passed": true
  }
}
```

It does **not** need to be stored as one document. A typical implementation would use separate logical tables.

#### Traces

```text
trace_id
created_at
model
task_id
dataset
reward
passed
metadata JSON
```

#### Spans or steps

```text
trace_id
span_id
parent_span_id
sequence_number
span_type
started_at
duration_ms
tool_name
input JSON
output JSON
attributes JSON
```

The API then reconstructs the tree:

```text
trace
 ├─ model call
 │   ├─ tool call
 │   └─ tool call
 └─ grader
```

A trace is normally a **tree or directed graph of spans**, not inherently a nested database document. Each span carries identifiers such as:

```text
trace_id
span_id
parent_span_id
```

The nesting is reconstructed using `parent_span_id`.

### Why not store every trace as one JSON document?

You certainly can, especially initially:

```sql
CREATE TABLE traces (
    trace_id UUID,
    created_at DateTime64,
    payload JSON
)
ENGINE = MergeTree
ORDER BY (created_at, trace_id);
```

But exclusively storing one opaque document makes common questions harder:

```text
Which browser tool produced the most failures?

What is the p95 duration of model calls by model version?

Which task types have more than three failed tool calls?

Show all spans where reward < 0.5 and tool_name = browser.
```

At scale, you usually want **one row per span or event**, with frequently queried attributes promoted to columns.

The original nested trace can still be preserved:

```text
Structured span rows       → fast querying
Raw trace JSON             → exact replay/debugging
Large files/screenshots    → object storage
```

### ClickHouse can still store nested structures

ClickHouse offers several relevant data types:

```text
JSON
Map(String, String)
Array(...)
Tuple(...)
Nested(...)
```

For example:

```sql
CREATE TABLE spans
(
    timestamp DateTime64(3),
    trace_id UUID,
    span_id UUID,
    parent_span_id Nullable(UUID),

    span_type LowCardinality(String),
    model LowCardinality(String),
    tool_name LowCardinality(String),

    duration_ms UInt32,
    reward Nullable(Float32),

    input JSON,
    output JSON,
    attributes Map(String, String)
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(timestamp)
ORDER BY (trace_id, timestamp, span_id);
```

The important distinction is:

> **Nested JSON is the trace’s interchange and presentation format. ClickHouse rows are its analytical storage representation.**

### Likely architecture for a HUD/W&B-style product

A solid implementation would be:

```text
                         ┌────────────────────┐
SDK / evaluation runner ─► ingestion service │
                         └─────────┬──────────┘
                                   │
                ┌──────────────────┼──────────────────┐
                ▼                  ▼                  ▼
           PostgreSQL         ClickHouse          S3 storage
           ──────────         ──────────          ──────────
           users              traces              screenshots
           projects           spans               recordings
           task configs       scores              large outputs
           permissions        token usage         raw trace JSON
           job state          latency             datasets
```

For your own system, I would probably model it as:

- **PostgreSQL:** models, experiments, datasets, task definitions, users and jobs
- **ClickHouse:** rollouts, spans, tool calls, token usage, timing and evaluation scores
- **S3:** full raw JSON, screenshots, recordings, checkpoints and large outputs

So your observation about nested JSON does not argue against ClickHouse. In fact, **W&B’s confirmed use of ClickHouse suggests that the nested trace UI and a columnar analytical backend fit together very naturally**.

## 3. Parquet in object storage versus ClickHouse

**Source question:** how about using plain parquet files rather than clickhouse? What would be pros vs. cons? The option is to have the parquet files stored at an object storage type of storage, which, effectively, gives an indefinite amount of storage for the traces data even in the analytical format

Yes, **Parquet files in object storage can absolutely be the primary analytical store** for AI evaluation traces. For a system dominated by append-only data, long retention, batch analysis, and relatively relaxed query latency, it may be more economical and architecturally cleaner than keeping everything in ClickHouse.

But I would avoid thinking of it as merely “plain Parquet files.” At meaningful scale, the stronger design is usually:

```text
Object storage
    └── Iceberg table
          └── Parquet data files

Queried through:
    DuckDB / ClickHouse / Trino / Spark / Athena
```

Parquet is the **physical file format**. Iceberg, Delta Lake, or another table format supplies the table-level metadata and management that loose Parquet files lack.

### The essential distinction

| | Parquet in object storage | ClickHouse |
|---|---|---|
| What it is | Durable analytical files | Running analytical database |
| Storage cost | Very low | Higher |
| Storage capacity | Effectively object-storage scale | Limited by provisioned storage, although object-backed configurations exist |
| Query engine | Must be supplied separately | Built in |
| Query latency | Usually seconds to minutes | Often milliseconds to seconds |
| Ingestion visibility | Usually delayed until files close/commit | Near-real-time |
| Operational overhead | Low initially; file/table management emerges later | Database cluster must be managed |
| Interactive trace UI | More difficult | Natural fit |
| Long-term archive | Excellent | Usually not the cheapest option |

Parquet is column-oriented, compressed, supports complex nested structures, and was designed with nested data in mind. That makes it technically well suited to JSON-like AI traces. ([Parquet](https://parquet.apache.org/?utm_source=chatgpt.com))

### Advantages of Parquet on object storage

#### 1. Very low storage cost

This is the most obvious advantage.

You can retain:

- every evaluation;
- complete prompts and outputs;
- span metadata;
- grader results;
- tool-call details;
- token usage;
- raw trace attributes;
- historical schema versions.

You do not have to aggressively expire older traces simply to control database disk usage.

For AI traces, where the raw data may become valuable later for regression analysis or retraining, this is a strong advantage.

#### 2. No database storage lock-in

The data remains in an open format that many systems can read:

- DuckDB;
- ClickHouse;
- Spark;
- Trino;
- Athena;
- BigQuery external tables;
- Snowflake external tables;
- Python libraries such as PyArrow and Polars.

Parquet is widely supported across languages and analytics systems. ([Parquet](https://parquet.apache.org/docs/overview/?utm_source=chatgpt.com))

You could begin with DuckDB, later introduce ClickHouse, and still keep the same underlying data.

#### 3. Excellent for batch evaluation

Suppose you need to run:

```text
Compare all GPT-5 evaluation runs from the last six months,
grouped by task, dataset version and evaluator version.
```

That is a classic analytical scan. A query engine can read only the required Parquet columns rather than the whole trace payload.

Parquet also stores statistics that engines can use for row-group and page pruning; its optional page index can support skipping pages that cannot satisfy a filter. ([Parquet](https://parquet.apache.org/docs/file-format/pageindex/?utm_source=chatgpt.com))

#### 4. Nested trace representation fits well

Parquet can represent:

```text
trace
  run_id
  model
  input
  spans[]
    span_id
    parent_span_id
    type
    input
    output
    attributes
  evaluation
    scores[]
```

You do not necessarily need to flatten everything into strings or manually serialize the structure as JSON. Parquet was explicitly designed to support complex nested structures using repetition and definition levels. ([Parquet](https://parquet.apache.org/docs/overview/motivation/?utm_source=chatgpt.com))

That said, whether storing a whole trace as one nested row is the best query model is a separate question.

#### 5. Easy separation of hot and cold data

You can keep all historical data in object storage indefinitely while loading only recent or frequently queried traces into ClickHouse.

```text
Last 30–90 days       → ClickHouse
Complete history      → Parquet/object storage
```

This is often the best economic compromise.

ClickHouse can read and write Parquet directly and query Parquet in object storage, so introducing ClickHouse later does not require abandoning the file-based source of truth. ([ClickHouse](https://clickhouse.com/docs/integrations/data-formats/parquet?utm_source=chatgpt.com))

### Disadvantages of plain Parquet files

#### 1. Parquet is not a database

A Parquet file does not give you:

- a global transaction log;
- table snapshots;
- atomic multi-file commits;
- row-level updates;
- schema coordination;
- file compaction;
- indexing across the dataset;
- concurrency control;
- automatic partition management.

Once you have thousands or millions of files, you need a layer that knows:

```text
Which files belong to this table?
Which ones are current?
Which schema applies?
Which files were replaced?
Which snapshot is valid?
```

This is why I would strongly prefer **Iceberg over loose Parquet directories**.

Iceberg adds schema evolution, hidden partitioning, partition-layout evolution, snapshots and time travel over Parquet files. ([Apache Iceberg](https://iceberg.apache.org/docs/1.10.0/?utm_source=chatgpt.com))

#### 2. Small-file problems

Traces arrive individually, but Parquet works best when data is written into reasonably large files.

Writing one Parquet file per trace would be a poor design:

```text
trace-1.parquet
trace-2.parquet
trace-3.parquet
...
trace-10000000.parquet
```

The problems include:

- object-store request overhead;
- slow file listing;
- excessive metadata;
- inefficient query planning;
- tiny row groups;
- poor compression;
- excessive file-open operations.

You normally need buffering:

```text
Incoming spans
      ↓
Kafka / queue / buffer
      ↓
Batch every 30–300 seconds
      ↓
128–512 MB Parquet file
```

The exact target depends on the query engine and workload, but the general requirement is to batch records and periodically compact files.

That means traces may not become queryable immediately.

#### 3. Higher query latency

With ClickHouse, querying recent traces usually means reading indexed or sorted database parts that are already attached to the running server.

With Parquet, the engine may need to:

1. retrieve table metadata;
2. identify candidate files;
3. fetch footer metadata;
4. open multiple objects;
5. read selected row groups;
6. reconstruct nested columns;
7. aggregate the result.

Partition pruning and file statistics reduce this work, but they do not remove it.

ClickHouse can query Parquet directly and has highly optimized support for it, but native ClickHouse tables will generally provide better latency and more effective data skipping for repeated interactive queries. ClickHouse itself distinguishes direct lake querying from data loaded into native tables. ([ClickHouse](https://clickhouse.com/docs/cloud/get-started/cloud/use-cases/data_lake_and_warehouse?utm_source=chatgpt.com))

#### 4. Poor fit for trace-by-trace navigation

A trace UI frequently performs queries such as:

```text
Load trace 01J...
Load all spans for this trace
Load its parent and children
Load the previous failed run
Show similar failures
Filter spans by tool name
```

If files are partitioned by date, finding one trace by `trace_id` might require consulting a catalog/index or scanning many file metadata entries.

Object storage does not naturally provide:

```text
SELECT * FROM spans WHERE trace_id = 'abc'
```

with database-like low latency.

You may need a separate trace index:

```text
trace_id → Iceberg partition/file location
```

or a small database containing trace headers and lookup information.

#### 5. Updates and late-arriving events are awkward

AI traces are often not completely immutable at first:

- the trace begins;
- spans arrive over time;
- the final output arrives;
- evaluators run later;
- human labels are added;
- a corrected score is produced;
- metadata is enriched.

Parquet files are immutable in practice. Updating one row normally means writing a new file or delete/update metadata.

Iceberg handles this substantially better than loose files, including snapshot-based table operations and copy-on-write or merge-on-read update modes. ([Apache Iceberg](https://iceberg.apache.org/docs/latest/configuration/?utm_source=chatgpt.com))

Still, frequent tiny updates remain much less natural than writing to PostgreSQL or ClickHouse.

#### 6. Operational complexity moves rather than disappears

Object storage itself is simple, but a production lakehouse often adds:

- ingestion buffers;
- Parquet writers;
- file compaction;
- catalog service;
- Iceberg metadata;
- schema management;
- query engine;
- caching;
- cleanup of orphaned files;
- monitoring of failed commits.

So “Parquet removes the database” is only partly true. It often replaces one visible database with several lighter data-lake components.

### Whole trace per row versus one span per row

This matters significantly.

#### Option A: one nested trace per row

```text
trace_id
timestamp
model
dataset
spans: array<struct<...>>
scores: array<struct<...>>
metadata
```

**Advantages**

- natural representation;
- straightforward trace retrieval;
- complete trace stays together;
- fewer joins;
- convenient for replay and export.

**Disadvantages**

- querying individual spans is more cumbersome;
- one very large trace can produce an oversized row;
- traces cannot be written until complete without later rewriting;
- partial updates are difficult;
- analytics over tools and spans require unnesting arrays.

#### Option B: one span per row

```text
trace_id
span_id
parent_span_id
timestamp
span_type
model
tool_name
duration
input
output
attributes
```

**Advantages**

- efficient span-level analytics;
- append-friendly;
- easy grouping by tool, model, error or evaluator;
- straightforward duration and percentile calculations;
- maps well to OpenTelemetry.

**Disadvantages**

- retrieving a complete trace requires collecting multiple rows;
- trace-level metadata may be repeated;
- tree reconstruction happens in the application or query.

#### Practical model

Use multiple tables:

```text
traces
    one row per trace

spans
    one row per span

scores
    one row per evaluator result

artifacts
    one row per external payload
```

These can all be Parquet-backed Iceberg tables.

Keep the original nested trace JSON or Parquet record as an immutable replay artifact when exact reproduction matters.

### Plain Parquet versus Iceberg

I would draw a fairly firm line here.

#### Plain Parquet directories are reasonable when

- one writer controls all data;
- data is append-only;
- schemas rarely change;
- queries are mostly offline;
- the dataset is not yet enormous;
- you can derive file paths predictably;
- occasional manual compaction is acceptable.

For example:

```text
s3://traces/
  project_id=123/
    event_date=2026-07-28/
      hour=20/
        part-0001.parquet
        part-0002.parquet
```

#### Iceberg becomes valuable when

- there are multiple writers;
- schemas evolve;
- data arrives late;
- files need compaction;
- queries must see consistent snapshots;
- partitions need to change over time;
- records may be corrected or deleted;
- reproducibility matters.

Iceberg’s hidden partitioning allows the query to filter by logical columns without requiring clients to know the physical partition layout; it can also evolve that layout while old and new layouts coexist. ([Apache Iceberg](https://iceberg.apache.org/docs/latest/evolution/?utm_source=chatgpt.com))

For evaluation data, snapshotting and time travel are particularly useful:

```text
Reproduce the metrics as they appeared
before the evaluator logic changed.
```

### Where ClickHouse remains stronger

ClickHouse is preferable when the application itself is an interactive trace exploration product.

Examples:

- dashboards refresh every few seconds;
- traces appear immediately after execution;
- users filter millions of spans interactively;
- p95 and p99 latency charts are recalculated frequently;
- arbitrary tag filtering is common;
- you need fast trace-ID lookup;
- many users issue concurrent queries;
- alerts run continuously.

That is likely part of why W&B Weave uses ClickHouse: the product is not only storing traces but actively serving an observability-style UI.

### Where Parquet is stronger

Parquet plus object storage may be preferable when:

- trace retention is effectively permanent;
- most analysis is asynchronous;
- evaluation jobs run periodically;
- query latency of several seconds is acceptable;
- traces are usually consumed in large batches;
- cost matters more than interactive responsiveness;
- data portability is important;
- you already operate a warehouse/lakehouse query engine.

A team performing overnight evaluation comparisons may not need an always-on ClickHouse cluster at all.

### Cost and performance are not binary

A useful design is to treat Parquet as the source of truth and ClickHouse as a serving layer:

```text
                         ┌──► ClickHouse
Trace ingestion ────────┤    Recent/searchable traces
                         │
                         └──► Iceberg + Parquet
                              Complete durable history
```

Or:

```text
Iceberg + Parquet
       │
       ├── DuckDB for local/ad-hoc analysis
       ├── Trino/Athena for distributed batch queries
       └── ClickHouse for interactive product queries
```

This avoids duplicating large raw artifacts unnecessarily while still offering fast application behavior.

### My recommendation for your case

Based on your description—AI training/evaluation traces, nested JSON-like records, and a desire for indefinite analytical retention—I would seriously consider:

```text
Canonical storage:
    Iceberg tables backed by Parquet in object storage

Tables:
    traces
    spans
    evaluation_results
    artifacts

Immediate/control state:
    PostgreSQL

Initial query engine:
    DuckDB for development or smaller datasets
    Trino/Athena/warehouse for distributed analysis

Optional serving layer:
    ClickHouse only for recent data and interactive UI
```

I would **not** make loose Parquet files the only abstraction unless this is an early-stage or tightly controlled system. Iceberg provides important database-like table semantics without giving up object-storage economics.

#### Decision rule

Choose **Parquet/Iceberg only** when:

```text
seconds-level queries are acceptable
+ ingestion may be delayed
+ analytics are mostly batch-oriented
+ low storage cost is the priority
```

Choose **ClickHouse plus Parquet/Iceberg** when:

```text
you need interactive trace exploration
+ near-real-time visibility
+ many concurrent analytical queries
+ indefinite low-cost retention
```

So yes: your idea is sound, and for the **system of record**, Parquet in object storage may actually be better than ClickHouse. ClickHouse becomes valuable primarily as a **query-serving and observability layer**, not necessarily as the only or permanent location of the trace data.

## 4. Apache Iceberg technical and operational deep dive

**Source question:** Tell me the technical details needed for running Apache Iceberg. What are the technologies used in developing Iceberg, what are the minimal and recommended system requirements for running one in productions, and what other relevant things I should know for making a decision whether it is a technology I'm willing to adopt in my stack or not

#### The most important point first

**Apache Iceberg is not a database server that you install and “run.”** It is an open table format plus client libraries that define how analytical tables are represented, versioned, committed, and discovered.

A production Iceberg platform normally consists of:

```text
Query / processing engines
  Spark, Flink, Trino, etc.
          │
          ▼
Iceberg catalog
  REST, AWS Glue, Hive Metastore,
  JDBC, Nessie, vendor catalog...
          │
          ▼
Object storage
  S3, GCS, Azure Blob, MinIO, HDFS...
          │
          ▼
Data and metadata files
  Parquet / ORC / Avro data
  Avro manifests + JSON metadata
```

Iceberg itself does not execute SQL, schedule jobs, allocate compute, or provide storage. Your query engine and catalog do those jobs. ([Apache Iceberg](https://iceberg.apache.org/?utm_source=chatgpt.com))

The current official Java release documented as of July 2026 is **Iceberg 1.11.0**, released in May 2026. ([Apache Iceberg](https://iceberg.apache.org/docs/latest/?utm_source=chatgpt.com))

---

### 1. What Iceberg actually stores

An Iceberg table generally contains four layers.

##### Data files

Usually:

- Parquet
- ORC
- Avro

For most modern lakehouse installations, **Parquet is the normal default**.

##### Delete files

With Iceberg format v2 and later, row-level changes may be represented using:

- position delete files
- equality delete files

Depending on the engine and configuration, updates and deletes may use either:

- **copy-on-write**: rewrite affected data files
- **merge-on-read**: write delete files and reconcile them during reads

The default delete mode in the Java implementation remains copy-on-write unless configured otherwise. ([Apache Iceberg](https://iceberg.apache.org/docs/latest/configuration/?utm_source=chatgpt.com))

##### Manifest files

Avro files containing information about groups of data and delete files, including:

- file paths
- partition values
- row counts
- column statistics
- lower and upper bounds
- file sequence information

These allow engines to avoid listing an entire bucket or scanning every Parquet footer.

##### Table metadata

JSON metadata files describe:

- schemas
- partition specifications
- snapshots
- sort orders
- table properties
- current snapshot
- references such as branches and tags

A snapshot points to a manifest list, which points to manifests, which point to data files.

This metadata tree is the core of Iceberg's transactional model. ([Apache Iceberg](https://iceberg.apache.org/spec/?utm_source=chatgpt.com))

---

### 2. Technologies used to develop Iceberg

#### Main implementation: Java

The primary Apache Iceberg project is written predominantly in **Java**.

It is:

- built with Gradle
- built using Java 17 or Java 21
- divided into modules for core functionality, storage integrations and compute-engine adapters
- tested using regular unit tests and Docker-based integration tests

The project produces dedicated runtime artifacts for different Spark and Flink versions because those engines have incompatible APIs between major versions. ([GitHub](https://github.com/apache/iceberg?utm_source=chatgpt.com))

For example, you may encounter artifacts such as:

```text
iceberg-spark-runtime-3.5_2.12
iceberg-spark-4.0_2.13
```

The suffixes matter:

- Spark version must match
- Scala binary version must match
- Iceberg runtime version should be explicitly pinned

Do not casually use a generic “latest” JAR in production.

#### PyIceberg: Python

Apache also maintains **PyIceberg**, a native Python implementation of the Iceberg specification.

It can be useful for:

- metadata operations
- Python-native ingestion
- lightweight table reads and writes
- integration with Arrow-based systems
- avoiding a JVM for some workflows

PyIceberg is not merely a Python wrapper around the Java library. It implements Iceberg functionality natively. Its configuration can be supplied using YAML, environment variables or Python API parameters. ([PyIceberg](https://py.iceberg.apache.org/configuration/?utm_source=chatgpt.com))

However, you should validate feature parity before standardising on it. New table-format features and maintenance operations may appear in Java first.

#### Iceberg Rust

Apache also maintains an official **Rust implementation**.

It is useful for:

- Rust-native systems
- DataFusion and Arrow ecosystems
- lower-overhead services
- embedded Iceberg readers and writers

It follows a rolling minimum-supported-Rust-version policy. It is developing quickly, but its exact feature support should be checked against your required Iceberg format version and operations. As of 2026, some format-v3 capabilities are still being tracked separately in the Rust project. ([GitHub](https://github.com/apache/iceberg-rust?utm_source=chatgpt.com))

#### File and metadata formats

The Iceberg specification itself relies heavily on:

- JSON for top-level table metadata
- Avro for manifest lists and manifest files
- Parquet, ORC or Avro for data
- HTTP and OpenAPI for the REST Catalog protocol

Iceberg defines an official REST Catalog API for table and namespace operations. ([Apache Iceberg](https://iceberg.apache.org/rest-catalog-spec/?utm_source=chatgpt.com))

---

### 3. Minimal infrastructure

Because Iceberg is not a server, the true minimum depends on what you intend to do.

#### Absolute development minimum

For a local proof of concept:

```text
1 machine
4 CPU cores
8–16 GB RAM
20–100 GB local disk
Java 17 or 21
Spark local mode or PyIceberg
Local filesystem or MinIO
Hadoop catalog or REST/JDBC catalog
```

You could technically use less, but 8 GB becomes uncomfortable once Spark, a catalog and object-storage emulator run together.

A simple local stack might be:

```text
Spark
Iceberg runtime JAR
MinIO
A lightweight catalog
PostgreSQL, if the catalog requires it
```

Docker Compose is entirely adequate for learning and integration tests.

#### Minimal credible production stack

A small but real production environment should normally have:

```text
Object storage:
  S3, GCS, Azure Blob or production-grade MinIO

Catalog:
  Managed catalog or at least two catalog instances
  Backed by a reliable transactional database where applicable

Compute:
  At least one supported query or processing engine

Maintenance:
  Scheduled compaction, snapshot expiration and orphan cleanup

Observability:
  Metrics, logs, failed-commit alerts and storage-cost monitoring
```

Using a single local filesystem and a single Spark server is not a production lakehouse, even though it may technically operate.

---

### 4. Practical production sizing

The following numbers are **starting points**, not official Iceberg requirements. Iceberg's own resource consumption is generally minor compared with query and rewrite jobs.

#### Catalog service

For a stateless REST catalog service:

| Deployment | CPU | Memory | Replicas |
|---|---:|---:|---:|
| Small production | 2 vCPU | 4–8 GB | 2 |
| Moderate workload | 4 vCPU | 8–16 GB | 2–3 |
| Heavy commit traffic | 8+ vCPU | 16+ GB | 3+ |

The important capacity metric is not table size. It is usually:

- commits per second
- concurrent writers
- number of tables
- metadata request rate
- authentication overhead
- database latency

A table containing 500 TB may create very little catalog load if it receives one daily commit. A 20 GB streaming table committing every few seconds may create much more catalog and metadata pressure.

#### Catalog database

For a JDBC-backed or REST catalog backed by PostgreSQL:

```text
2–4 vCPU
8–16 GB RAM
SSD-backed storage
point-in-time recovery
automated backups
high availability where table availability is critical
```

The JDBC database must provide atomic transactions and suitable isolation because Iceberg commits depend on an atomic metadata-pointer change. The official JDBC Catalog documentation requires a database capable of atomic transactions and read-serializable isolation. ([Apache Iceberg](https://iceberg.apache.org/docs/latest/jdbc/?utm_source=chatgpt.com))

The catalog database does **not** store your actual analytical data. It usually stores references, namespaces, table registrations and transactional state.

#### Query engine

This is where most resources go.

##### Small Trino installation

A reasonable initial production configuration:

```text
1 coordinator:
  4–8 vCPU
  16–32 GB RAM

2–3 workers:
  8–16 vCPU each
  32–64 GB RAM each
```

That is not an Iceberg requirement. It is a practical lower-end analytical cluster.

##### Small Spark environment

For scheduled ETL and maintenance:

```text
Driver:
  2–4 vCPU
  4–8 GB RAM

Executors:
  4–8 vCPU
  16–32 GB RAM each
  Start with 2–4 executors
```

Large compaction jobs can require considerably more temporary compute than routine queries.

##### Flink

For streaming ingestion, sizing depends primarily on:

- incoming event rate
- checkpoint frequency
- table partitioning
- commit interval
- state size
- number of output files

Flink's Iceberg sink supports exactly-once semantics. ([Apache Iceberg](https://iceberg.apache.org/docs/latest/flink-writes/?utm_source=chatgpt.com))

#### Object storage

Capacity is usually not the problem. Request patterns are.

You need:

- high read throughput
- reliable multipart uploads
- adequate request-rate capacity
- consistent permissions
- lifecycle rules
- encryption and key management
- sensible separation between production and temporary locations

For on-premises MinIO, use a genuinely distributed setup. A single MinIO container is suitable for development, not for a production analytical platform.

---

### 5. Catalog choices

The catalog is one of the most consequential decisions.

#### Managed cloud catalog

Examples include AWS Glue and equivalent managed services.

**Advantages**

- minimal operational burden
- native cloud identity integration
- high availability handled by the provider
- easy integration with cloud query engines

**Disadvantages**

- cloud coupling
- API costs and rate limits
- behaviour may differ between providers
- portability can be less clean than the table format itself suggests

Iceberg officially supports AWS Glue as a catalog, including optimistic locking for atomic commits. ([Apache Iceberg](https://iceberg.apache.org/docs/latest/aws/?utm_source=chatgpt.com))

#### REST Catalog

A REST catalog gives engines a standard HTTP interface rather than linking every engine directly to a particular database or metastore.

**Advantages**

- better client/server separation
- language-neutral protocol
- centralised authorization and credential vending
- easier multi-engine integration
- potentially easier catalog migration

**Disadvantages**

- another production service
- availability becomes important
- implementation choice still matters
- authentication and authorization require design

Iceberg defines the protocol, but you still need an implementation or managed service. ([Apache Iceberg](https://iceberg.apache.org/rest-catalog-spec/?utm_source=chatgpt.com))

#### JDBC Catalog

Uses tables in a relational database.

**Advantages**

- relatively simple
- PostgreSQL is familiar and well understood
- fewer specialist components
- appropriate for modest deployments

**Disadvantages**

- you operate the database
- direct database access from clients may be awkward
- authorization is less elegant than a service-based catalog
- scaling and cross-region usage need care

#### Hive Metastore

**Advantages**

- mature
- widely supported
- useful in an existing Hadoop environment

**Disadvantages**

- operationally dated for a greenfield stack
- adds Hadoop/Hive heritage you may not otherwise want
- schema synchronization with non-Iceberg Hive consumers can create complications

For a new non-Hadoop platform, I would not select Hive Metastore merely because it is historically common.

#### Nessie

Nessie adds:

- Git-like branches and tags
- multi-table transaction workflows
- versioned catalog state

Iceberg's normal transaction boundary is one table. Nessie is attractive where you need coordinated versioning across many tables or branch-based data engineering. ([Apache Iceberg](https://iceberg.apache.org/docs/latest/nessie/?utm_source=chatgpt.com))

It is also another system to understand and operate, so I would only add it when those semantics have actual business value.

---

### 6. Concurrency and transaction model

Iceberg uses **optimistic concurrency**.

A writer:

1. reads the current table metadata;
2. prepares new data and metadata files;
3. creates a new metadata tree;
4. attempts to atomically replace the catalog's current metadata pointer;
5. retries or fails if another writer committed first.

Readers continue to use their original snapshot, so they see a consistent table even while new commits occur. ([Apache Iceberg](https://iceberg.apache.org/docs/latest/reliability/?utm_source=chatgpt.com))

This gives Iceberg:

- snapshot isolation
- atomic table-level commits
- concurrent reads and writes
- time travel
- rollback
- serializable behaviour for supported operations

But it does **not automatically mean arbitrary multi-table ACID transactions**.

That distinction matters. A transformation updating five Iceberg tables can leave the first three updated and the final two unchanged if the process fails, unless your catalog or architecture adds multi-table transaction semantics.

---

### 7. Production maintenance is mandatory

This is the area most frequently underestimated.

#### Small-file compaction

Streaming and frequent batch ingestion tend to produce many small files.

Small files cause:

- excessive object-store requests
- larger manifests
- slower planning
- poor Parquet scan efficiency
- increased memory use in the query engine

You need regular data-file rewriting or compaction.

A common target file size is somewhere around:

```text
256 MB–1 GB per Parquet file
```

Around 512 MB is a reasonable starting target for many analytical workloads, but workload, engine and object store should determine the final value.

#### Manifest rewriting

Even when data files are healthy, manifests can become fragmented. Periodic manifest rewriting reduces planning overhead.

#### Snapshot expiration

Every commit creates a snapshot. Without expiration:

- metadata accumulates
- old data files remain retained
- storage consumption grows
- planning can become slower
- time-travel history grows indefinitely

Iceberg provides explicit snapshot-expiration operations. ([Apache Iceberg](https://iceberg.apache.org/docs/latest/maintenance/?utm_source=chatgpt.com))

#### Orphan-file deletion

Failed jobs can upload files that never become part of a committed snapshot.

Iceberg provides orphan-file cleanup, but it must be used conservatively. A retention interval that is too short can delete files belonging to an in-progress job. ([Apache Iceberg](https://iceberg.apache.org/docs/latest/maintenance/?utm_source=chatgpt.com))

#### Streaming metadata control

High-frequency streaming commits rapidly create metadata files. The official documentation recommends controlling commit frequency, expiring old snapshots and automatically deleting old metadata. ([Apache Iceberg](https://iceberg.apache.org/docs/latest/spark-structured-streaming/?utm_source=chatgpt.com))

A production installation normally needs maintenance schedules such as:

```text
Compaction:
  hourly, daily or based on file-count thresholds

Manifest rewrite:
  daily or weekly

Snapshot expiration:
  daily

Orphan cleanup:
  daily or weekly, with a safe age threshold

Statistics refresh:
  after major writes or compaction
```

The exact frequency should be table-specific.

---

### 8. Operational concerns

#### Version compatibility

You must maintain a compatibility matrix for:

- Iceberg library version
- Spark version
- Scala version
- Flink version
- Trino version
- catalog implementation
- table format version
- supported features such as deletion vectors or format v3

“Supports Iceberg” does not mean “supports every Iceberg operation.”

One engine may support:

- reading v2 tables
- writing append operations
- reading position deletes

but not:

- equality deletes
- branch writes
- certain schema changes
- v3 data types
- all maintenance procedures

Treat compatibility as a release-management concern.

#### Schema evolution

Iceberg tracks columns by stable field IDs rather than relying only on column names or positions.

This enables safer:

- renaming
- reordering
- adding fields
- deleting fields
- nested-schema evolution

It is a major advantage over manually managed directories of Parquet files.

#### Partition evolution

Iceberg supports hidden partitioning and partition-spec evolution.

Applications query logical columns:

```sql
WHERE event_timestamp >= ...
```

They do not need to know that data is physically partitioned by:

```text
days(event_timestamp)
```

You can later change the partition strategy without rewriting all historical data immediately.

#### Sort order

Partitioning is not your only physical-layout decision.

Within files, sorting by commonly filtered or joined fields can significantly improve:

- data skipping
- compression
- locality
- merge performance

Compaction should ideally respect the table's sort order.

#### Object-store consistency and commit correctness

Do not improvise commits by directly replacing files in S3.

Use:

- a proper Iceberg catalog
- a supported locking or atomic-commit implementation
- the normal Iceberg APIs

The documentation specifically notes that uncatalogued Hadoop-style tables on S3 may require a lock manager because S3 does not provide filesystem-style atomic mutual exclusion. ([Apache Iceberg](https://iceberg.apache.org/docs/latest/catalog-properties/?utm_source=chatgpt.com))

#### Security

Plan separately for:

- catalog authentication
- catalog authorization
- object-store credentials
- per-table or per-prefix access
- encryption at rest
- encryption in transit
- credential rotation
- audit logging
- row and column security at the query-engine layer

Object-store access and catalog access must agree. A user who can discover a table but cannot read its objects will fail. A user who can directly read arbitrary bucket paths may bypass governance implemented only in the catalog or query engine.

#### Disaster recovery

Back up or replicate:

- catalog state
- catalog database
- object-storage metadata and data
- encryption keys
- catalog configuration
- access policies

The table's metadata files are in object storage, but losing the catalog pointer or registrations can still cause serious operational work.

Object-versioning and cross-region replication can help, but they do not replace a tested recovery procedure.

---

### 9. Performance characteristics

Iceberg performs particularly well when:

- data volumes are large
- queries are analytical
- data is append-heavy or batch-updated
- multiple engines need shared access
- schema and partition layouts evolve
- object storage is the primary persistence layer
- time travel and reproducibility matter

It is not intrinsically faster than a database in every scenario.

Query performance depends heavily on:

- file sizes
- partition transforms
- sort order
- column statistics
- manifest quality
- query-engine implementation
- object-store latency
- cache strategy
- frequency and type of row-level updates

Iceberg improves metadata pruning and consistency. It cannot compensate for ten million 20 KB Parquet files.

---

### 10. When Iceberg is a good choice

Iceberg is a strong fit when you need several of these:

- tens of terabytes to petabytes of analytical data
- inexpensive object-storage retention
- Spark, Flink, Trino or several engines accessing the same data
- mutable analytical tables
- schema evolution
- partition evolution
- snapshot-based reproducibility
- time travel
- decoupled storage and compute
- open-format portability
- batch and streaming convergence

For your earlier use case involving **AI training and evaluation traces**, Iceberg becomes particularly attractive when:

- traces accumulate indefinitely in object storage
- individual trace records contain nested structures
- you need SQL analysis over large historical periods
- append rates are high
- schemas evolve as new evaluators and model metadata are added
- you want several analytical tools to share the data
- you require safe compaction and partition evolution

---

### 11. When Iceberg may be the wrong choice

It may be excessive when:

- you have only a few hundred GB
- one application is the only reader
- your access pattern is primarily key-by-ID lookup
- you need millisecond point reads
- you need high-rate transactional updates to individual rows
- you do not already operate Spark, Flink or Trino
- a single database such as PostgreSQL or ClickHouse satisfies the workload
- you are unwilling to operate scheduled data maintenance

Iceberg is also not a direct replacement for:

- PostgreSQL as an OLTP system
- Kafka as an event transport
- ClickHouse as a low-latency serving database
- Elasticsearch/OpenSearch as a text-search engine
- a metadata catalog or governance platform

It can complement these systems.

---

### 12. Iceberg versus plain Parquet files

This is likely the central decision in your case.

Plain Parquet in object storage gives you:

- inexpensive storage
- portability
- columnar analytics
- simple ingestion
- minimal platform complexity

Iceberg adds:

- atomic commits
- consistent snapshots
- schema evolution
- partition evolution
- reliable concurrent writers
- row-level deletes and updates
- time travel
- metadata-based file pruning
- table history
- standardized table discovery
- safe compaction semantics

The cost is:

- a catalog
- version management
- maintenance jobs
- more metadata
- greater operational complexity
- stricter writer discipline
- engine compatibility management

A useful dividing line is:

> Plain Parquet manages files. Iceberg manages a table made from files.

If you are only archiving immutable traces and occasionally scanning complete daily partitions, plain Parquet may be sufficient.

If you need concurrent ingestion, evolving schemas, corrections, compaction, multiple query engines and a dependable table abstraction, Iceberg provides substantial value.

---

### 13. A sensible small production architecture

For a team that wants an open, manageable stack:

```text
Storage:
  S3-compatible object storage

Catalog:
  Iceberg REST Catalog
  backed by PostgreSQL

Interactive SQL:
  Trino

Batch ingestion and maintenance:
  Spark

Streaming, only if required:
  Kafka + Flink

File format:
  Parquet with Zstandard compression

Orchestration:
  Airflow, Dagster, Argo Workflows or your existing scheduler

Monitoring:
  Prometheus/Grafana
  object-storage metrics
  catalog and PostgreSQL metrics
  Spark/Trino metrics
```

For an AWS-first organisation seeking minimal operations:

```text
Storage:
  S3

Catalog:
  AWS Glue

Batch:
  AWS Glue jobs, EMR or Spark on Kubernetes

Interactive:
  Athena or Trino

Streaming:
  Flink where necessary

Governance:
  Lake Formation or query-engine policies
```

For a compact initial deployment, I would not introduce Spark, Flink, Trino, Kafka and Nessie simultaneously. Start with:

```text
Object storage
+ catalog
+ one compute engine
+ scheduled maintenance
```

Add other engines only when a concrete workload needs them.

---

### 14. My adoption assessment for your stack

Given your background with AWS, Kubernetes, PostgreSQL and analytical infrastructure, the technology itself is well within your operational capabilities.

The more important question is whether its benefits justify introducing a **lakehouse control plane**.

I would rate it this way:

| Criterion | Assessment |
|---|---|
| Technical maturity | High |
| Open ecosystem | High |
| Object-storage suitability | Excellent |
| Multi-engine support | Excellent, but version-sensitive |
| Operational simplicity | Medium to low |
| Small-data suitability | Low to medium |
| Append-only trace suitability | Good |
| Mutable/evolving trace suitability | Excellent |
| Low-latency query serving | Not its primary role |
| Long-term storage economics | Excellent |
| Lock-in risk | Lower than proprietary warehouse formats |
| Maintenance burden | Material |

#### My practical recommendation

For an AI-evaluation-trace platform, I would adopt Iceberg when at least two or three of these are already true:

- expected data volume is in the multi-terabyte range and growing;
- multiple services or engines will read and write the data;
- schemas will evolve frequently;
- traces need correction, deletion or reprocessing;
- historical snapshots and reproducibility matter;
- files require ongoing compaction;
- you want object storage to be the durable source of truth.

I would stay with plain partitioned Parquet initially when:

- the dataset is append-only;
- one ingestion service controls all writes;
- queries are relatively infrequent;
- partitions can be treated as immutable;
- the team wants the smallest possible operational footprint.

A particularly sensible adoption path is:

```text
Phase 1:
  Parquet on object storage
  disciplined directory layout
  sufficiently large files
  explicit schema management

Phase 2:
  Register or migrate those files into Iceberg
  add a catalog
  introduce snapshot and compaction workflows

Phase 3:
  Add multiple engines, streaming and advanced governance
```

However, if you already know that you need concurrent writers, updates, time travel or multi-engine access, starting with plain Parquet may merely postpone the table-management problems. In that case, beginning with Iceberg is cleaner.

## Appendix A: Original research prompts

### Conversation 1: Best datastore for AI traces

1. What is the best data store for storing AI models training evaluation traces data?
2. We were looking at tools like hud.ai and wandb, and was wondering what data store they use for their traces, which apparently look like nested JSONs
3. how about using plain parquet files rather than clickhouse? What would be pros vs. cons? The option is to have the parquet files stored at an object storage type of storage, which, effectively, gives an indefinite amount of storage for the traces data even in the analytical format

### Conversation 2: Apache Iceberg Technical Details

1. Tell me the technical details needed for running Apache Iceberg. What are the technologies used in developing Iceberg, what are the minimal and recommended system requirements for running one in productions, and what other relevant things I should know for making a decision whether it is a technology I'm willing to adopt in my stack or not

## Appendix B: Resolved source index

The following links were embedded in the ChatGPT answers' citation metadata and are listed here for convenient review.

1. [Introduction | ClickHouse Docs](https://clickhouse.com/docs/use-cases/observability/introduction)
2. [Traces](https://opentelemetry.io/docs/concepts/signals/traces/)
3. [Parquet - Apache Software Foundation](https://parquet.apache.org/)
4. [Documentation: 18: 8.14. JSON Types](https://www.postgresql.org/docs/current/datatype-json.html)
5. [Set up a self-managed W&B Weave instance](https://docs.wandb.ai/weave/guides/platform/weave-self-managed)
6. [W&B Self-Managed deployment overview](https://docs.wandb.ai/platform/hosting/hosting-options/self-managed)
7. [HUD Documentation — Evaluations and RL Environments.](https://docs.hud.ai/building/environments-as-data)
8. [Overview - Apache Parquet](https://parquet.apache.org/docs/overview/)
9. [Page Index - Apache Parquet](https://parquet.apache.org/docs/file-format/pageindex/)
10. [Motivation - Apache Parquet](https://parquet.apache.org/docs/overview/motivation/)
11. [Working with Parquet in ClickHouse](https://clickhouse.com/docs/integrations/data-formats/parquet)
12. [Introduction - Apache Iceberg™](https://iceberg.apache.org/docs/1.10.0/)
13. [Data warehousing | ClickHouse Docs](https://clickhouse.com/docs/cloud/get-started/cloud/use-cases/data_lake_and_warehouse)
14. [Configuration - Apache Iceberg™](https://iceberg.apache.org/docs/latest/configuration/)
15. [Schema evolution - Apache Iceberg](https://iceberg.apache.org/docs/latest/evolution/)
16. [Apache Iceberg - Apache Iceberg™](https://iceberg.apache.org/)
17. [Documentation - Apache Iceberg](https://iceberg.apache.org/docs/latest/)
18. [Spec - Apache Iceberg™](https://iceberg.apache.org/spec/)
19. [Apache Iceberg](https://github.com/apache/iceberg)
20. [Configuration](https://py.iceberg.apache.org/configuration/)
21. [apache/iceberg-rust](https://github.com/apache/iceberg-rust)
22. [REST Catalog Spec - Apache Iceberg™](https://iceberg.apache.org/rest-catalog-spec/)
23. [JDBC - Apache Iceberg™](https://iceberg.apache.org/docs/latest/jdbc/)
24. [Flink Writes - Apache Iceberg™](https://iceberg.apache.org/docs/latest/flink-writes/)
25. [AWS - Apache Iceberg™](https://iceberg.apache.org/docs/latest/aws/)
26. [Nessie - Apache Iceberg™](https://iceberg.apache.org/docs/latest/nessie/)
27. [Reliability - Apache Iceberg™](https://iceberg.apache.org/docs/latest/reliability/)
28. [Maintenance - Apache Iceberg™](https://iceberg.apache.org/docs/latest/maintenance/)
29. [Spark Structured Streaming - Apache Iceberg](https://iceberg.apache.org/docs/latest/spark-structured-streaming/)
30. [Catalog properties - Apache Iceberg™](https://iceberg.apache.org/docs/latest/catalog-properties/)
