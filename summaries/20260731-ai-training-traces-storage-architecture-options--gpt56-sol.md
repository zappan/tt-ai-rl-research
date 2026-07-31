# AI Training Traces Storage: Architecture Options

## Problem definition

AI training, evaluation, and agent traces must support two different demands: fast investigation of recent activity and economical analysis across a large historical record. The data arrives incrementally, evolves over time, and may include sensitive or large artifacts that require durable retention and controlled deletion. A suitable architecture must balance interactive performance with cost, portability, consistency, governance, and operational effort.

## Preferred direction

- **Iceberg and Parquet on object storage:** canonical analytical trace history.
- **ClickHouse:** query Iceberg directly first; add native tables only for measured performance bottlenecks.
- **PostgreSQL:** transactional control-plane state.
- **DuckDB:** bounded, governed analysis from researcher workstations.

The working principle is to preserve one open, durable source of truth and duplicate analytical data only when benchmarks show that direct lake queries cannot meet product requirements.

```text
Trace and evaluation producers
             |
             v
    Buffered ingestion layer
             |
     +-------+--------------------------------+
     |                                        |
     v                                        v
PostgreSQL                            Object storage
control-plane state                    |
                              +---------+----------+
                              |                    |
                              v                    v
                         Large artifacts     Iceberg + Parquet
                                             canonical history
                                                    |
                              +---------------------+------------------+
                              |                     |                  |
                              v                     v                  v
                       ClickHouse direct     Optional native     DuckDB or other
                            queries          ClickHouse tables   governed engines
```

This direction keeps the complete history economical, portable, reproducible through snapshots, and accessible to several tools. ClickHouse can query the Iceberg tables without first copying all trace data; native storage remains an acceleration path when direct queries miss explicit service objectives, followed by a complete hot layer only if selective acceleration is insufficient.

The preference is conditional because direct lake access is not equivalent to native ClickHouse MergeTree storage. Catalog access, object-store latency, snapshot freshness, file layout, and pruning determine real performance, so benchmarks must decide how much native acceleration the product needs.

## Architecture options

| Option | Expected strength | Principal limitation |
|---|---|---|
| Direct ClickHouse-on-Iceberg | One open analytical copy with a capable shared SQL engine | Lake latency, freshness, concurrency, and write-path compatibility must be proven |
| ClickHouse-native canonical storage | One ingestion path and predictable interactive performance without Parquet or Iceberg | Database-specific format, concentrated recovery responsibilities, and no direct multi-engine access |
| Selective ClickHouse acceleration | Fast known workloads without copying the complete history | Requires policies and pipelines for selecting and refreshing accelerated data |
| Replicated ClickHouse hot layer | Most predictable low-latency product experience | Highest duplication, synchronization, and operational cost |
| PostgreSQL plus object storage | Smallest initial operational footprint | Limited analytical scale and interactive multidimensional querying |

### Option 1: ClickHouse queries Iceberg directly

Iceberg tables backed by Parquet remain canonical, while ClickHouse connects through the catalog and queries them in place. This preserves storage openness, keeps ingestion and snapshots independent from the serving engine, and may satisfy research, batch analysis, and moderately interactive product workloads without a second complete copy.

Its main risk is performance variability. Iceberg queries still involve metadata planning, remote reads, and the available file and column statistics. New traces may not be visible until files and snapshots commit. The production write path and selected catalog must also be validated separately; ClickHouse read support does not establish that ClickHouse should own every Iceberg write and maintenance task.

### Option 2: ClickHouse as the canonical analytical store

ClickHouse can own the canonical structured traces, spans, evaluation results, and searchable metadata in native MergeTree tables. PostgreSQL retains control-plane state, while object storage holds large artifacts, backups, and potentially cold native parts; Parquet and Iceberg are not required. This remains viable at petabyte scale: compression, object-backed storage, tiering, and horizontal scaling make data volume alone an insufficient reason to reject it. The design offers one ingestion path, immediate query visibility, native indexing, and the fewest analytical platform components.

The tradeoff is a ClickHouse-specific canonical format. Other engines and researcher workstations must query through ClickHouse or use exported extracts, while retention, recovery, mutations, and migration depend on one database platform. Prefer it when the interactive product dominates and that coupling is acceptable. Evaluate it on portability, independent access, retention economics, and recovery concentration—not assumed scale limits.

### Option 3: selectively accelerate workloads in ClickHouse

ClickHouse continues to query the complete Iceberg history directly, while selected datasets are loaded into native tables. Candidates include recent partitions, a trace-ID index, common dashboard aggregates, or frequently filtered typed columns.

This approach provides native sorting and indexing for known bottlenecks without copying the full history. It is the preferred next step when only a bounded group of direct-lake queries misses its objectives.

The tradeoff is routing and lifecycle complexity. The platform must decide which queries use Iceberg, which use accelerated tables, how the native data refreshes, and what freshness users can expect. It becomes less attractive if arbitrary exploration requires nearly every recent field.

### Option 4: continuously replicated hot and cold layers

Every committed trace is written or replicated to native ClickHouse tables for a recent window while Iceberg retains the complete history. Product queries use native ClickHouse storage; historical and cross-engine analysis uses Iceberg.

This provides the most predictable trace-navigation, filtering, and dashboard performance. It also creates the greatest operational burden: duplicated data, a second ingestion path, freshness monitoring, reconciliation, backfills, and recovery rules for partial failures. Adopt it only when direct querying and selective acceleration cannot meet broader interactive latency and concurrency requirements.

### Option 5: PostgreSQL plus object storage

PostgreSQL owns application state and modest trace volumes, while object storage retains raw payloads and artifacts. Batched Parquet exports can support offline analysis without immediately introducing an Iceberg catalog or analytical database.

This is a sensible product-validation posture because it minimizes infrastructure and supports transactional updates. It is not the preferred long-term analytical design: PostgreSQL eventually absorbs scans and aggregations it was not selected to perform, while loose Parquet exports lack the reliable table semantics required by shared, evolving data.

## Shared assumptions and operating consequences

- **Data model:** Use related `traces`, `spans`, `evaluation_results`, and `artifacts` datasets linked through stable identifiers. Keep common dimensions typed, long-tail attributes flexible, and large or original payloads in object storage.

- **Ingestion and late data:** The writer or stream processor owns buffering. Use batched ClickHouse inserts or an Iceberg-compatible writer that creates appropriately sized Parquet files and commits them atomically. Prefer append-only facts for late scores, labels, and corrections. Iceberg manages files and commits; it is not an ingestion buffer.

- **Iceberg operations:** Iceberg still requires object storage, a catalog, compatible writers and readers, a query engine, compaction, snapshot expiration, metadata cleanup, orphan-file removal, and aligned permissions across the storage, catalog, and query layers.

- **ClickHouse operations:** ClickHouse-native storage requires replication, tested backup and restore, TTL or tiering policies, merge monitoring, and controlled mutations. Accelerated native tables additionally require refresh, query routing, freshness, and consistency management.

- **Research access:** DuckDB can query bounded Parquet or Iceberg data directly. With ClickHouse-canonical storage, researchers must use ClickHouse or exported extracts. Both paths require governed access to sensitive trace data.

## Decision framework

Validate the preferred direction in stages:

1. Establish representative volumes, file layouts, late-data behavior, retention horizons, and the shared span-oriented schema.
2. Benchmark direct ClickHouse-on-Iceberg against ClickHouse-native canonical tables using the same ingest and query workload.
3. Compare freshness, p50 and p95 trace lookup, high-cardinality filters, dashboards, historical scans, concurrency, compressed storage, replicas and backups, restore time, export portability, and researcher workflow friction.
4. If Iceberg remains canonical and only bounded workloads miss their objectives, test selective native acceleration before continuous hot replication.

Use the results to determine whether open storage, independent researcher access, reproducible snapshots, and separation from one engine outweigh the interactive performance and smaller platform surface of ClickHouse-native storage. Test catalog compatibility, updates and deletes, backup and restore, access control, migration, and disaster recovery alongside query performance.
