# Executive Summary: AI Trace Storage

**Date:** 2026-07-29

AI trace data creates two competing demands. Product teams need to inspect recent traces quickly, navigate model and tool calls, and investigate failures. At the same time, the organization needs to retain a large and growing history of prompts, responses, scores, labels, and artifacts for evaluation, audit, replay, and model improvement.

No single datastore serves both needs optimally. A fast analytical database is well suited to interactive investigation but can be unnecessarily expensive as the permanent home of all historical data. Object storage is inexpensive and portable, but it does not provide the low-latency experience expected from an observability product.

The recommended solution is a layered architecture:

- **PostgreSQL** manages transactional application data such as users, projects, experiments, permissions, configurations, and workflow state.
- **Object storage with Parquet and Apache Iceberg** becomes the durable system of record for complete trace history. This provides economical retention, open data access, reliable snapshots, and support for evolving schemas.
- **ClickHouse** is added as a serving layer only when the product requires near-real-time trace visibility, interactive filtering, frequent dashboards, or high query concurrency.

This design separates durable storage from query serving. It keeps the complete history in an open, low-cost format while allowing recent or frequently accessed traces to be optimized for speed.

The main tradeoff is operational complexity. Iceberg requires a catalog, query engine, file compaction, metadata cleanup, monitoring, and recovery procedures. ClickHouse adds another production database. These components should therefore be introduced in response to concrete requirements rather than all at once.

The recommended adoption path is to start with PostgreSQL and object storage—or disciplined Parquet for a controlled, append-only workload. Add Iceberg when multiple writers, schema changes, corrections, reproducible snapshots, or multi-engine access become necessary. Add ClickHouse when measured latency and concurrency requirements justify a dedicated interactive layer.

**Decision:** adopt object storage with Iceberg/Parquet as the long-term trace foundation, retain PostgreSQL for transactional state, and treat ClickHouse as an optional performance layer. This provides the best balance of cost, responsiveness, portability, and future scale.

For supporting analysis and implementation considerations, see [AI Trace Storage: Problem, Options, and Recommended Architecture](20260729-ai-trace-storage-synthesis--gpt56-sol.md). The underlying research was consolidated from the repository sources and was not independently revalidated.
