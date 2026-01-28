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

### Block Diagram
```mermaid
flowchart LR
    subgraph Clients
        SDK[SDKs<br/>Batch + Retry]
        API[REST/gRPC API]
    end
    subgraph Control
        Router[Routing Service<br/>Shard Map + Consistent Hash]
        Config[Metadata DB<br/>shard assignments]
    end
    subgraph DataPlane
        WQ[Write Worker<br/>UPSERT/CAS]
        PG[(Postgres/Citus<br/>Primary Nodes)]
        RR1[(Read Replica 1)]
        RR2[(Read Replica 2)]
    end
    subgraph Background
        TTL[TTL & Compaction<br/>Workers]
        CDC[CDC Stream<br/>Event Bus / S3]
    end
    SDK --> API --> Router --> WQ --> PG
    Router --> RR1
    Router --> RR2
    TTL --> PG
    PG --> CDC
    Router <-- Config
```

### Detailed Component Design
1. **Routing Layer**
   - Maintains shard map with `(tenant_id hash -> shard_id)` persisted in a metadata DB so all stateless API nodes share the same source of truth.
   - Caches mappings aggressively (5–10 minutes) with version stamps; background watcher invalidates cache when topology changes.
   - Supports partial rebalancing by migrating a tenant or hash range to a new shard while dual-writing during cutover.
2. **API/Service Layer**
   - Exposes REST and gRPC endpoints; every call attaches `tenant_id` and `idempotency_key`.
   - Connection pools per shard keep max concurrency bounded; prepared statements avoid repeated parse/plan overhead.
   - Implements optimistic concurrency control by requiring `If-Match: version` headers on updates.
3. **Storage Shards**
   - Each shard runs Postgres with table partitioned by `tenant_hash`. Extensions like Citus/Vitess provide logical sharding and distributed queries.
   - WAL archiving + point-in-time recovery enable fast restores; base backups taken daily.
4. **TTL / Compaction Workers**
   - Scan `ttl` index in batches (`WHERE ttl < now()` LIMIT 10k) to delete expired rows and emit metrics `expired_rows_per_min`.
   - Optionally migrate cold keys to cheaper storage (S3) using COPY TO + delete.
5. **Change Data Capture (CDC) & Analytics**
   - Logical replication slots or Debezium connectors stream mutations into Kafka/S3 for search indexing or auditing.
   - CDC also powers warm caches or regional replicas without hammering primaries.
6. **Observability & Guardrails**
   - Per-shard dashboards showing `qps`, `p99`, `connection_utilization`, `bloat`, `autovacuum_lag`.
   - Automatic throttling when shard CPU or lock contention crosses thresholds; hints clients to retry with exponential backoff.

### Data & Consistency Flows
- **Write Path**
  1. Client sends `PUT /kv/{key}` with latest `version`.
  2. Router chooses shard -> service executes `UPDATE ... WHERE version = :old_version` (CAS). If 0 rows affected, returns `409 Conflict`.
  3. Successful write triggers CDC event and invalidates corresponding cache entries.
- **Read Path**
  1. Router selects nearest healthy replica; uses prepared `SELECT value, version FROM kv WHERE tenant_id=? AND key=?`.
  2. Fallback to primary if replica lag exceeds SLA; optional quorum read for strong consistency.
- **Range/Scan**
  - POST `/kv/query` compiles filters into SQL with `LIMIT/OFFSET` and pushes down pagination tokens (encoded `tenant_id + cursor`).
- **TTL Expiry**
  - Worker reads `ttl` index, deletes expired rows, emits metrics, and optionally publishes deletion event for downstream caches/search.

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

## Interview Preparation Aids
### High-Level Talking Points
- Emphasize why SQL engines remain attractive (transactions, tooling) even for KV workloads.
- Explain partitioning choices (tenant hashing vs. consistent hashing) and how to migrate shards safely.
- Highlight consistency story (read replicas eventual, CAS for correctness) and resilience (CDC, PITR, failover).

### Sample Questions & Answers
1. **Q:** *How do you prevent a single tenant from overwhelming one shard?*  
   **A:** Track per-tenant qps + storage in metadata; when thresholds exceed limits, initiate live rebalancing: introduce a fresh shard, update routing map to dual-write (old+new) for targeted tenants, drain backlog, then deactivate the old mapping. Implement tenant-level rate limiting and localized caches to smooth bursts.
2. **Q:** *What happens if optimistic concurrency keeps failing under high contention?*  
   **A:** Detect repeated 409 responses and switch to server-side transactional retry with backoff, or escalate to advisory locks for that key. Another tactic is to split hot keys into subkeys (e.g., `key#device`) and aggregate asynchronously.
3. **Q:** *How do you guarantee TTL accuracy without hurting performance?*  
   **A:** Use index scans with bounded batch size plus a time-wheel scheduler so expirations are spread evenly. Combine lazy expiration (on read) with proactive sweeps. Surface `ttl_lag_seconds` metrics; alert if lag > SLA and consider partitioning TTL tables by date.
4. **Q:** *How would you extend this design for multi-region active-active?*  
   **A:** Partition tenants to home regions and use logical replication for async copies. For global tenants, introduce per-region primaries with conflict-free replicated data types or global transaction managers only where strong consistency is mandatory. The router becomes geo-aware and includes locality hints in tokens.

### Mock Whiteboard Prompts
- Draw the block diagram above and walk interviewers through write/read flows in <5 minutes.
- Justify storage choices by comparing Postgres+Citus vs. DynamoDB or Spanner for the same workload.
- Discuss an incident response: replica lag spikes; describe detection, mitigation (route reads to primaries, throttle writers), and long-term fixes.
