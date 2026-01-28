# 08. Designing a Scalable SQL-backed KV Store

## Problem Overview
- Provide key-value semantics (CRUD, TTL, versioning) while leveraging relational features such as transactions and SQL tooling.

## Functional Requirements
- CRUD APIs with atomic read-modify-write transactions per key.
- TTL + background eviction, optimistic concurrency via version column.
- Partitioning/sharding for horizontal scale and read replicas for fan-out.

## Non-Functional Goals
- Target 5 ms p99 read latency on warm shard, 20 ms writes.
- Sustain 200k ops/sec with linear horizontal scaling.
- Online schema changes without downtime.

## Architecture Overview
- Logical table `(tenant_id, key, version, value, ttl, updated_at)` partitioned by tenant+hashed key.
- Router service maps keys to shards via consistent hashing / Citus coordinator.
- Write path uses UPSERT + `WHERE version = ?` for CAS semantics; read path hits replicas via prepared statements.

## Data Design & APIs
- Value stored as JSONB or bytea; optional secondary indexes per tenant.
- APIs: `PUT /kv/{key}`, `GET /kv/{key}`, `POST /kv/query` for range scans, `DELETE /kv/{key}`.
- TTL sweeper job reads upcoming expirations via index on `ttl`.

## Implementation Plan
1. Choose Postgres+Citus (or Vitess) deployment topology; set up shard routing metadata.
2. Implement service layer with prepared statements, connection pooling, and CAS wrappers.
3. Build TTL/compaction workers and metrics for tombstones.
4. Add replication + failover automation; test read replicas + promotion.
5. Offer SDKs + client libraries with batching, retries, and instrumentation.

## Testing & Validation
- Run workload generator with skewed keys to ensure shard balance.
- Validate CAS by racing updates from multiple clients.
- Perform failover drills ensuring router updates quickly.

## Operational Considerations
- Monitor shard size, replica lag, bloat; set auto-rebalancing thresholds.
- Document hot-key mitigation strategies (local caches, key hashing, queueing writes).
