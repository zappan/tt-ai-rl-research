# AI Trace Datastore Executive Summary

**Compiled:** 2026-07-29  
**Model:** GPT-5.5  
**Source:** `20260729-ai-trace-datastore-summary--gpt55.md`

## Executive answer

AI trace storage should not be treated as one database decision. Training and
evaluation traces combine several different workloads:

- fast lookup of individual traces for debugging;
- high-volume analytics across spans, models, datasets, tools, evaluators, and
  runs;
- long-term retention of raw prompts, outputs, artifacts, and historical
  results;
- evolving schemas as models, tools, and evaluation methods change;
- late-arriving scores, labels, corrections, and enrichments.

No single datastore naturally serves all of these needs. The best architecture
separates responsibilities:

```text
PostgreSQL       Application control plane
ClickHouse       Hot interactive trace analytics
Object storage   Raw payloads, artifacts, and durable storage
Parquet          Columnar analytical file format
Apache Iceberg   Table metadata, snapshots, and commits over Parquet
```

The recommended production answer is a hybrid design:

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
control plane           recent traces    raw payloads and
metadata                and analytics    Iceberg/Parquet history
```

This architecture gives the platform responsive trace debugging, scalable
analytics, low-cost retention, and an open durable history.

## The problem

An AI trace records how a model or agent produced an outcome. It may include
prompts, model calls, tool calls, retrieved documents, parent-child spans,
timings, token usage, costs, evaluator scores, human labels, errors,
screenshots, recordings, and raw outputs.

Products often show this as nested JSON, but that should be understood as the
API or UI representation. It does not have to be the physical storage model.

The physical model should usually split traces into related datasets:

```text
traces      one row per trace or run
spans       one row per model call, tool call, evaluator step, or event
scores      one row per evaluator result or label
artifacts   one row per stored raw payload or external object
```

The UI can reconstruct a nested trace from `trace_id`, `span_id`, and
`parent_span_id`. This makes analytical questions much easier:

- Which tool caused the most failures?
- What is p95 latency by model version?
- Which dataset regressed after a prompt change?
- Which evaluator version changed pass rates?

Store the original nested trace or raw export as an immutable replay artifact
when exact reproduction matters.

## Recommended storage roles

| Layer | Responsibility | Why |
|---|---|---|
| PostgreSQL | Users, projects, permissions, experiments, configs, workflow state | It is transactional, familiar, and good for mutable application metadata. |
| ClickHouse | Recent traces, spans, scores, latency, token usage, costs, dashboards | It is strong for append-heavy, high-cardinality analytical queries and interactive trace UIs. |
| Object storage | Prompts, outputs, screenshots, recordings, datasets, checkpoints, raw JSON | It provides low-cost durable retention for large or infrequently queried data. |
| Parquet | Analytical files in object storage | It is compressed, columnar, portable, and efficient for historical scans. |
| Apache Iceberg | Table management over Parquet | It adds snapshots, atomic commits, schema evolution, partition evolution, and consistent reads. |

This split avoids using ClickHouse as an expensive permanent archive, avoids
using PostgreSQL for massive analytical scans, and avoids pretending that a
folder of Parquet files is a production database.

## Decision guide

Use **PostgreSQL plus object storage** when:

```text
the product is early
+ trace volume is modest
+ transactions matter
+ operational simplicity matters most
```

Use **ClickHouse** when:

```text
users need interactive trace exploration
+ traces must appear quickly
+ dashboards and filters must be responsive
+ analytics span many high-cardinality dimensions
```

Use **Iceberg plus Parquet** when:

```text
long-term retention matters
+ object-storage cost matters
+ batch historical analytics are common
+ multiple engines or teams need the same data
+ schemas, partitions, or records will evolve
+ reproducible historical snapshots matter
```

Use **ClickHouse plus Iceberg/Parquet** when the platform needs both:

```text
low-latency product queries
+ complete low-cost analytical history
```

That hybrid is the strongest general-purpose architecture.

## Practical adoption path

Start small:

```text
PostgreSQL + JSONB
Object storage for raw payloads and artifacts
```

Even in the first version, model trace data around spans and stable identifiers.
Avoid designing the system around one opaque JSON blob per trace.

As volume grows, add ClickHouse for recent interactive trace analytics:

```text
Recent or frequently queried traces -> ClickHouse
Raw payloads and artifacts          -> Object storage
Control-plane state                 -> PostgreSQL
```

When durable analytical history becomes important, add Iceberg tables backed by
Parquet:

```text
Complete trace/span/score history -> Iceberg + Parquet on object storage
Interactive recent data           -> ClickHouse
Application metadata              -> PostgreSQL
Large raw artifacts               -> Object storage
```

Operate the lakehouse deliberately. Batch writes, avoid tiny files, compact
regularly, monitor the catalog, expire snapshots under policy, clean orphaned
files, and protect sensitive raw payloads.

## Final recommendation

The best possible solution is not to choose one datastore for everything. It is
to assign each part of the workload to the system that fits it:

```text
PostgreSQL owns application state.
ClickHouse serves the interactive trace product.
Object storage keeps the heavy raw truth.
Parquet stores analytical history efficiently.
Iceberg makes that history reliable, consistent, and portable.
```

For an early system, PostgreSQL plus object storage is enough. For a production
AI evaluation platform, the target architecture should be PostgreSQL,
ClickHouse, object storage, and Iceberg/Parquet working together.
