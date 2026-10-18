# Performance

Performance is a feature. Design for the P99, not the average. Measure before you optimize — intuition is almost always wrong about where the time is going.

## Rules

- **Profile before optimizing.** Guessing the hot path wastes days. Use flame graphs, APM traces, or `EXPLAIN ANALYZE` first.
- **Kill N+1 queries.** The single biggest latency sink in service code. Prefer joins, batch fetches, or `IN (...)` over per-row lookups.
- **Paginate every list endpoint.** Unbounded result sets eventually OOM a pod or saturate the DB. Default to cursor-based pagination for large tables.
- **Index on query predicates, not every column.** Each extra index slows writes and bloats storage. Add an index only when a known query needs it.
- **Project only the columns you need.** `SELECT *` into a wide DTO wastes network, memory, and GC. Return narrow projections for list APIs.
- **Cache read-heavy, idempotent data.** But decide the invalidation strategy *before* adding the cache — a cache without an eviction plan is a bug factory.
- **Set connection and thread pool sizes explicitly.** Defaults are rarely right for your workload. Size them against downstream capacity, not up.
- **Batch outbound calls.** One request with 100 items beats 100 requests with one item on every axis — latency, CPU, DB connections.
- **Avoid synchronous chains of remote calls.** Latency compounds. Fan out in parallel, or move the work async.
- **Keep payloads small.** Compress responses, trim unused fields, use binary protocols where it matters (gRPC, Protobuf).

## Red flags

- Loops that query the DB.
- APIs that return "all records."
- Caches with no TTL and no invalidation.
- Thread pool / connection pool set to the framework default in production.
- Latency dashboards that show averages only (no P95/P99).
