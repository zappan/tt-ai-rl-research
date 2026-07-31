# ClickHouse as the Canonical Store for AI Training Traces

## Problem definition

The current trace-storage direction makes Iceberg and Parquet the canonical analytical record and uses ClickHouse as a query or acceleration layer. A credible alternative is to remove the lakehouse table layer and make ClickHouse's native storage the canonical home of structured traces, spans, evaluation results, and searchable metadata. The decision is whether this simpler, performance-oriented architecture remains sustainable as retained trace data grows, and what is lost when the canonical format belongs to one database engine.

## Conclusion

ClickHouse-native canonical storage is technically viable for very large AI trace datasets, including petabyte-scale deployments. Data volume alone is not a sound reason to reject it. ClickHouse is designed for high-ingest, append-heavy analytical workloads, compresses columnar data efficiently, scales horizontally, and can place native MergeTree parts on object storage. Parquet and Iceberg are therefore optional rather than technically required.

The more consequential tradeoff is architectural. A ClickHouse-canonical design concentrates ingestion, retention, querying, recovery, and schema management in one database-specific storage format. It gives the simplest path to immediate, interactive analytics but weakens direct multi-engine access, workstation analysis, open-format portability, and separation between the durable record and its serving engine.

This option should be evaluated as a first-class alternative. If it is rejected, the reason should be retention economics, portability, independent research access, recovery posture, or unacceptable technology concentration—not an assumption that ClickHouse cannot hold the data.

## Proposed ClickHouse-canonical architecture

```text
Trace and evaluation producers
             |
             v
    Buffered ingestion layer
             |
     +-------+-----------------------------+
     |                                     |
     v                                     v
PostgreSQL                          ClickHouse MergeTree
control plane                 canonical structured trace data
                                           |
                                  queries, TTLs, backups,
                                  tiering, and retention

Object storage
  large prompts, responses, media, checkpoints, raw artifacts
  optional native ClickHouse cold parts and external backups
```

In this design:

- **ClickHouse** owns structured traces, spans, evaluation results, labels, frequently searched attributes, and any current analytical views.
- **PostgreSQL** owns users, projects, permissions, experiment definitions, configurations, and workflow state.
- **Object storage** owns large or opaque artifacts that should not be duplicated inside analytical rows. It may also hold ClickHouse-native cold parts and backups.
- **Parquet and Iceberg** are absent from the primary architecture. They may be introduced later for exports, interchange, independent archival access, or migration.

Eliminating Parquet does not eliminate object storage. ClickHouse Cloud stores native data parts on object storage, and self-managed ClickHouse can use storage policies to move older native parts to object-backed volumes. The physical format remains ClickHouse MergeTree parts rather than open Parquet files.

## Viability at large scale

ClickHouse is explicitly designed for distributed, petabyte-scale analytics and high-cardinality observability data. Its sparse primary indexes, sorted parts, column pruning, compression codecs, parallel execution, sharding, and replication address the same workload shape as AI traces. ClickHouse describes deployments ranging to hundreds of servers and petabyte scale, while its observability stack uses ClickHouse as both storage and analytics for logs, metrics, and traces ([ClickHouse platform](https://clickhouse.com/clickhouse), [petabyte-scale observability](https://clickhouse.com/resources/engineering/managing-petabyte-scale-logs-without-sampling)).

Native storage is also no longer synonymous with locally attached disks. ClickHouse Cloud separates compute from object-backed storage and uses distributed caching to reduce remote-read latency. Self-managed installations can tier older parts from SSD to object storage with storage policies and TTL rules ([ClickHouse Cloud stateless compute](https://clickhouse.com/blog/clickhouse-cloud-stateless-compute), [observability storage tiering](https://clickhouse.com/resources/engineering/observability-cost-optimization-playbook)).

The sheer number of retained rows is therefore not the primary limitation. The material scale questions are:

- compressed bytes retained after codecs and schema choices;
- indexes, projections, replicas, and backup copies;
- background merge and mutation work;
- compute needed for ingest and interactive concurrency;
- frequency of wide historical scans;
- retention of large raw payloads that compress poorly;
- recovery time for a very large canonical service.

A representative benchmark and cost model are required because these factors depend on actual trace shape, sort order, query predicates, retention, and deployment model.

## Advantages

### One ingestion and consistency path

Events become queryable through the same system that stores them. There is no Iceberg commit followed by ClickHouse discovery, no second hot copy to reconcile, and no ambiguity over which representation is current. Late data can be appended as new facts, while current-state views can use ClickHouse table engines and materialized views suited to the chosen semantics.

### Predictable interactive performance

Native MergeTree tables provide ClickHouse sorting, sparse primary indexes, data-skipping indexes, projections, materialized views, and caches directly over the canonical data. This is the strongest topology for trace-by-ID lookup, recent-window filters, dashboards, and high query concurrency.

### Smaller platform surface

The system does not need an Iceberg catalog, a separate lakehouse writer, table-format compatibility testing, or Iceberg-specific snapshot and metadata maintenance. Managed ClickHouse can also absorb cluster sizing, replication, backups, and upgrades, reducing the practical operations gap between a database and a lakehouse.

### Retention can still use object storage

Native parts may live on object storage or move through hot and cold tiers. Retention rules can re-compress, move, aggregate, or delete data. This can make a ClickHouse-only analytical design considerably more economical than an architecture that assumes all native data must remain replicated on premium local SSDs.

## Long-term risks and limitations

### Database-specific canonical format

MergeTree parts are not Parquet. DuckDB, Spark, Trino, and other engines cannot independently query the canonical files. They must use ClickHouse as a service boundary or consume exported data. A future migration requires an active export rather than attaching another engine to the same open tables.

### Researcher independence

Researchers can query ClickHouse through SQL clients, Python libraries, or controlled extracts, but they lose the ability to point a workstation engine directly at the canonical history. If offline, reproducible, or tool-independent analysis is important, scheduled Parquet exports may reappear—partially undoing the simplicity benefit.

### Recovery and historical reproducibility

ClickHouse provides replication and backup mechanisms, and ClickHouse Cloud supports automated and external backups. Those backups can be retained in customer-controlled object storage, but full external backups of large services consume time, storage, and compute ([ClickHouse Cloud backups](https://clickhouse.com/cloud), [external backups](https://clickhouse.com/blog/introducing-external-backups-on-clickhouse-cloud)).

This is different from an Iceberg table's open snapshot graph and multi-engine time travel. Reproducing an analytical table as it appeared at a historical point depends on ClickHouse retention, backup, or explicit versioned facts rather than an independently queryable table snapshot.

### Mutations, deletions, and backfills

MergeTree data is stored as immutable parts that are merged in the background. Lightweight updates and deletes reduce the cost of small changes, but broad mutations and backfills can still rewrite affected parts and consume I/O and compute. Trace models should remain append-oriented where possible, especially for late scores and labels ([ClickHouse columnar storage](https://clickhouse.com/resources/engineering/what-is-columnar-storage)).

### Technology and service concentration

The canonical data, serving path, retention mechanism, and much of disaster recovery depend on ClickHouse. A database outage or operational error can affect both the source of truth and the product query path. Managed ClickHouse reduces day-to-day operations but increases dependence on service capabilities, pricing, regional recovery, and export paths.

### Large artifacts remain external

ClickHouse should not become the default container for recordings, images, checkpoints, or multi-megabyte raw payloads. Keeping those objects separately means the architecture still has two durability domains and must coordinate permissions, retention, and deletion between database rows and object-store artifacts.

## Direct comparison with an Iceberg-canonical design

| Decision dimension | ClickHouse canonical | Iceberg/Parquet canonical |
|---|---|---|
| Interactive product queries | Strongest default | Depends on engine, caching, and file layout |
| Ingest-to-query freshness | Immediate or near-real-time | Usually gated by file and snapshot commits |
| Platform components | Fewer | Object store, catalog, writers, maintenance, and engines |
| Canonical format | ClickHouse-specific | Open and independently queryable |
| Multi-engine access | Through ClickHouse or exports | Direct through compatible engines |
| Researcher workstation access | Service query or extract | Direct access is possible with DuckDB and similar tools |
| Historical snapshots | Database facts and backups | Native Iceberg snapshots and time travel |
| Long-term storage | Native parts, tiering, replicas, and backups | Low-cost Parquet plus Iceberg metadata |
| Product and storage failure domains | More concentrated | Durable record separated from serving compute |
| Migration path | Requires export | Attach another compatible engine |

## When ClickHouse should be preferred

Choose ClickHouse as the canonical analytical store when most of the following are true:

- the product is primarily an interactive trace or observability system;
- recent and historical data both require responsive queries;
- ClickHouse is the accepted shared analytical interface;
- direct multi-engine file access is not a strategic requirement;
- researchers can work through ClickHouse or governed extracts;
- managed ClickHouse or mature self-managed operations are available;
- retention, backup, and recovery costs are acceptable at projected volume;
- the organization accepts active export as the migration path.

## When Iceberg should remain canonical

Prefer Iceberg and Parquet when several of these conditions dominate:

- indefinite cold retention represents most stored bytes;
- multiple engines or teams need direct access to the same history;
- reproducible open snapshots are a core requirement;
- researchers need independent local analysis;
- storage-format portability is strategically important;
- the durable record should survive replacement of the serving engine;
- query latency can tolerate lake access or selective acceleration.

## Recommended decision test

Benchmark two complete designs using the same representative trace corpus:

1. ClickHouse-native canonical tables with realistic indexes, codecs, retention, backups, and concurrency.
2. Iceberg/Parquet canonical tables queried directly by ClickHouse, with the same query set and freshness targets.

Measure compressed storage, replicas and backups, monthly compute, ingest-to-query freshness, p50 and p95 trace lookup, dashboards, historical scans, concurrent users, deletion and backfill behavior, restore time, export throughput, and researcher workflow friction. Model costs at the expected one-, three-, and five-year retention volumes.

The current working preference remains Iceberg/Parquet canonical storage because it preserves openness and independent access. ClickHouse-canonical storage is nevertheless a strong alternative and could become preferable if interactive performance and operational simplicity matter more than multi-engine portability and an independent open archive.

## Related repository research

- [AI Trace Storage: Problem, Options, and Recommended Architecture](20260729-ai-trace-storage-synthesis--gpt56-sol.md)
- [AI Trace Storage, Parquet, and Apache Iceberg Architecture](20260729-ai-trace-storage-parquet-and-iceberg-architecture--gpt56-sol.md)
- [AI Trace Datastores and Apache Iceberg](20260729-ai-trace-datastore-apache-iceberg-research--gpt56-terra.md)
