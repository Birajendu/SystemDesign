# 03. Practical Nuances of Connection Pools & DB Proxies

## Problem Overview
- Prevent databases from saturating via uncontrolled client connections while keeping latency predictable.

## Functional Requirements
- Provide centralized proxy tier exposing PostgreSQL/MySQL endpoints with transaction and session pooling modes.
- Autosize pools per service based on concurrency budgets.
- Offer visibility into wait times, queue depth, and leaked connections.

## Non-Functional Goals
- 99p latency < 10 ms for acquiring pooled connections under nominal load.
- Fail closed on authentication/tenant leakage; degrade gracefully when DB is slow.

## Architecture Overview
- Sidecars or shared proxies (pgBouncer/Envoy) handle TLS termination, auth, and pooling.
- Control plane service distributes pool configs, rotation keys, and health data.
- Backpressure layer enforces circuit breakers & adaptive throttling derived from DB metrics.

## Data Design & APIs
- Config schema: `(service, db_role, max_connections, max_queue, timeout_ms, priority)` stored in etcd.
- Observability API surfaces `pool_utilization`, `wait_time_histogram`, `transaction_time` per tenant.

## Implementation Plan
1. Inventory all services and calculate concurrency via Little’s Law (qps × latency).
2. Deploy pgBouncer/Envoy proxies with TLS + SNI-based routing; enable transaction pooling first.
3. Build config service + admin UI for safe pool changes with staged rollouts.
4. Instrument proxies with OpenTelemetry spans and Prometheus metrics.
5. Introduce guardrails: global connection budget enforcement, auto-kill zombie sessions, and chaos drills.

## Testing & Validation
- Run soak tests at 150% peak to confirm queueing thresholds.
- Simulate DB failover to ensure proxies detect and reroute quickly.
- Verify auth isolation by attempting cross-tenant connections.

## Operational Considerations
- Provide runbooks for resizing pools, draining proxies, and handling TLS cert rotation.
- Alert on queue saturation, increasing wait times, and DB-side `too many connections` errors.
