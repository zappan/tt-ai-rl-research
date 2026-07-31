# AI Trace Datastore: Executive Summary

## Decision

Use a **hybrid storage architecture** for AI training and evaluation traces:

```text
PostgreSQL                 Application and transactional state
Object storage + Parquet   Durable raw payloads and long-term history
Apache Iceberg             Reliable analytical tables over Parquet
ClickHouse                 Recent, interactive trace exploration when required
```

This is the best general solution because AI traces need fast diagnosis and economical long-term analysis. No single datastore is the best fit for both jobs.

PostgreSQL should own the application's transactional data. Object storage, Parquet, and Iceberg should provide the durable analytical record. ClickHouse should be added as the fast serving layer when the product requires interactive trace navigation and near-real-time analytics.

## The problem

An AI trace records how a model evaluation or agent run produced an outcome. It can include prompts, model outputs, tool calls, parent-child spans, retrieved context, timing, token usage, costs, evaluator scores, human annotations, screenshots, recordings, and errors.

Although a trace is often shown to users as a nested JSON document, it is not static. Spans can arrive independently, scores may be added later, labels can be corrected, and the schema changes as models, tools, and evaluators evolve. The system must support fast trace lookup, flexible analysis across many dimensions, low-cost long-term retention, reliable concurrent ingestion, and controlled schema change.

These needs create unavoidable tradeoffs:

- A running analytical database provides fast queries but is more expensive for indefinite retention.
- Object storage is economical and durable but is not naturally fast for trace-by-ID navigation.
- A nested JSON document is convenient for replay and APIs but inefficient for span-level analysis at scale.
- Parquet is an excellent file format but does not by itself manage a shared, evolving production table.

The architecture should assign each responsibility to the component designed for it rather than attempting to make one system handle every concern.

## Recommended architecture

```text
SDKs, training workers, and evaluation runners
                    |
                    v
        Trace ingestion and batching layer
                    |
      +-------------+-------------+
      |                           |
      v                           v
 PostgreSQL                 Object storage
 control-plane data         raw payloads and Parquet files
                                  |
                                  v
                            Iceberg tables
                                  |
                         +--------+--------+
                         |                 |
                         v                 v
                    Batch analysis     ClickHouse
                    and reporting      interactive recent traces
```

| Component | Primary responsibility | Do not use it as |
|---|---|---|
| PostgreSQL | Users, projects, experiments, permissions, configuration, and workflow state | The long-term high-volume trace analytics engine |
| Object storage | Raw prompts, outputs, screenshots, recordings, datasets, and archival data | A low-latency trace-serving database |
| Parquet | Compressed, portable, columnar analytical files | A table-management or transaction system |
| Apache Iceberg | Snapshots, schema and partition evolution, concurrent table commits, and maintenance semantics | A standalone database server or trace UI backend |
| ClickHouse | Fast trace search, dashboards, high-cardinality analysis, and recent operational data | The only durable archive for every raw payload |

The ingestion layer should publish a stable span/event model, ideally using OpenTelemetry-style identifiers: `trace_id`, `span_id`, `parent_span_id`, timestamps, status, attributes, and artifact references. It should batch writes rather than create a storage object for every span.

## Data model and storage policy

Store the analytical representation as several related datasets:

- `traces`: one row per trace, including trace-level metadata and outcome.
- `spans`: one row per model call, tool call, grader, or other event.
- `evaluation_results`: one row per evaluator score or judgement.
- `artifacts`: one row per externally stored payload, with its URI and content hash.

The `spans` table supports the questions that make an AI trace platform useful: which tool fails most often, how latency changes by model and prompt version, which evaluator changed pass rates, and where costs are increasing. The application rebuilds the visual trace tree from `trace_id` and `parent_span_id`.

Retain the original nested trace JSON separately for exact replay and debugging. Store large prompts, transcripts, images, recordings, and checkpoints in object storage rather than duplicating them in every analytical row. This preserves fidelity while keeping operational queries responsive and storage costs controlled.

## When to use each approach

**Start with PostgreSQL plus object storage** when the product is early, trace volumes are modest, and the team needs the smallest operational footprint. Use typed columns for stable values and flexible metadata only where needed. Design the event schema so traces can later flow into ClickHouse or Parquet without a redesign.

**Make Iceberg and Parquet the primary analytical record** when long retention, open access, several query engines, evolving schemas, data corrections, snapshots, or multiple writers are already known requirements. Iceberg is especially valuable when the data has become a shared production table rather than an immutable archive.

**Add ClickHouse** when users need traces to appear quickly, filter high-cardinality dimensions interactively, perform trace-by-ID lookup with low latency, or continuously calculate dashboards and alerts. In this model, ClickHouse serves the active product experience while the full history remains durable in Iceberg.

**Use plain Parquet directories only** for an early-stage or tightly controlled archive: one writer, append-only data, infrequent offline queries, stable schemas, and manageable manual compaction. Once the platform needs concurrent writers, late corrections, schema evolution, or consistent snapshots, Iceberg should replace loose-file management.

## Operating requirements

The important operational issue is not whether object storage can hold trace data. It can. The issue is managing it as a dependable table system:

- Buffer events and write sufficiently large Parquet files; never create one Parquet file per trace.
- Schedule Iceberg compaction, snapshot expiration, metadata cleanup, and orphan-file deletion.
- Treat the Iceberg catalog as production infrastructure: secure, monitor, and back it up.
- Plan for late scores and annotations as append-only facts where possible, with controlled updates when a current-state view is necessary.
- Govern schema changes and keep writer, catalog, and query-engine versions compatible.
- Apply access controls and deletion workflows to raw prompts and outputs, which may contain sensitive data.

## Recommended adoption path

1. **Prove the product:** use PostgreSQL for control data, object storage for raw artifacts, and a stable span schema. Archive events in batched Parquet files if needed.
2. **Establish the durable record:** introduce an Iceberg catalog, create the four analytical tables, select one query engine, and automate maintenance before the table grows large.
3. **Serve interactive workflows:** replicate recent or frequently queried spans into ClickHouse when measured product latency requires it. Keep the hot-data retention window based on actual access patterns.
4. **Expand selectively:** add streaming, semantic search, full-text search, or additional query engines only for demonstrated needs. Vector and text-search databases are secondary indexes, not trace systems of record.

## Final recommendation

For a production AI evaluation or observability platform, use **object storage with Parquet and Iceberg as the durable analytical foundation**, **PostgreSQL for transactional application state**, and **ClickHouse for recent interactive trace analysis**. Begin more simply when the product is small; introduce Iceberg and ClickHouse when the workload demonstrates their value. This design preserves data economically, keeps it portable, supports reproducible historical analysis, and provides a responsive trace experience.

## Related research

- [Detailed AI Trace Datastore Architecture Summary](20260729-ai-trace-datastore-architecture-summary--gpt56-terra.md)
- [AI Trace Storage, Parquet, and Apache Iceberg Architecture](20260729-ai-trace-storage-parquet-and-iceberg-architecture--gpt56-sol.md)
- [AI Trace Storage and Apache Iceberg Research Notes](20260729-ai-traces-iceberg-research--gpt55.md)
- [AI Trace Datastores and Apache Iceberg](20260729-ai-trace-datastore-apache-iceberg-research--gpt56-terra.md)
