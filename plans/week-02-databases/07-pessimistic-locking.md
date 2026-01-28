# 07. Pessimistic Locking on Relational Databases

## Problem Overview
- Guarantee serialized updates for high-value rows (payments, inventory) even under contention, using DB-provided locks.

## Functional Requirements
- Expose APIs that wrap `SELECT ... FOR UPDATE` or advisory locks with timeout + retry semantics.
- Provide monitoring of lock wait graphs and deadlocks.
- Allow per-tenant isolation so noisy neighbors cannot starve others.

## Non-Functional Goals
- Keep lock wait p95 under 150 ms under nominal load; deadlock rate <0.1% of transactions.
- Support multi-region deployments with deterministic failover semantics (no double-exec after failover).

## Architecture Overview
- Application services use lock-aware DAO layer that centralizes acquisition/release and includes tracing.
- Lock management dashboard reads from DB views (`pg_locks`, `information_schema.innodb_locks`).
- Optional coordination service issues distributed fencing tokens for multi-DB workflows.

## Data Design & APIs
- Lock request payload: `(resource_id, scope (row/table), lock_mode, timeout_ms, priority)`.
- Metadata tables store lock audit logs and conflict counts.
- API wrappers: `LockContext.runWithLock(resource, fn)` handling retries/backoff.

## Implementation Plan
1. Identify critical tables/indexes that require pessimistic locking and annotate ORM/queries.
2. Build lock helper library with metrics, tracing, and consistent retry envelope.
3. Create dashboards + alerts for lock waits, blocked queries, and deadlock reports.
4. Stress-test by replaying conflicting transactions; tune indexes and query plans.
5. Document fallback strategies (queue requests, degrade features) when lock contention spikes.

## Testing & Validation
- Run concurrency harness issuing thousands of conflicting writes; record throughput vs. contention.
- Force failovers to ensure locks are released and app handles retries.

## Operational Considerations
- Provide procedures for killing runaway transactions safely.
- Educate teams about lock scope/timeouts to avoid global locks in OLTP paths.
