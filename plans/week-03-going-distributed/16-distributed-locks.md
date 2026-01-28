# 16. Implementing Remote & Distributed Locks

## Problem Overview
- Offer mutual exclusion across services without relying on database row locks, ensuring safety even under network faults.

## Functional Requirements
- Support lock acquisition with TTL (leases), optional reentrancy, and read/write modes.
- Expose APIs for try-lock with timeout, blocking lock, and unlock with fencing token.
- Provide metrics for contention, queue length, and expired leases.

## Non-Functional Goals
- Lock acquisition latency < 20 ms within region; system sustains 50k lock ops/sec.
- Guarantee that expired leases cannot continue mutating state (use fencing tokens / monotonic counters).

## Architecture Overview
- Lock service backed by Redis (SET NX + PX) or DynamoDB conditional writes; optionally etcd for strong consistency.
- Client SDK handles retries with jitter, renewal heartbeats, and token validation.
- Optional fairness queue to avoid starvation.

## Data Design & APIs
- Lock record: `(lock_id, owner_id, fencing_token, expires_at, metadata)`.
- APIs: `POST /locks/{id}:acquire`, `POST /locks/{id}:renew`, `POST /locks/{id}:release`, `GET /locks/{id}`.

## Implementation Plan
1. Choose storage primitive; configure replication and persistence.
2. Build lock service that issues monotonic fencing tokens and enforces lease expiries.
3. Implement client libraries (gRPC/HTTP) with instrumentation + circuit breakers.
4. Add admin tooling for force release, lock inspection, and deadlock detection.
5. Integrate with downstream systems requiring locks (payments, migrations) and run canary rollouts.

## Testing & Validation
- Stress test with thousands of contending clients; ensure fairness + throughput.
- Simulate client crashes to confirm leases expire and next holder proceeds safely.
- Validate fencing tokens by intentionally reusing expired tokens; downstream should reject.

## Operational Considerations
- Monitor replication lag, TTL expiries, and CPU/memory headroom.
- Provide runbooks for cache flushes, node replacement, and TTL tuning.

## Tutorial Deep Dive
### Block Diagram
```mermaid
flowchart LR
    Clients --> SDK[Client SDK\n(Retry + Metrics)]
    SDK --> LockSvc[Lock Service API]
    LockSvc --> Store[(Lock Store\nRedis/Dynamo/etcd)]
    LockSvc --> Audit[Audit/Event Stream]
    Store --> Monitor[Metrics + Expiry Alerts]
    LockSvc --> Tokens[Fencing Token Generator]
```

### Design Walkthrough
- **Acquisition path:** Clients request locks via SDK; service issues monotonic tokens and persists owner metadata with TTL.
- **Renewal/expiry:** Heartbeat jobs renew leases; expired locks trigger events so dependent systems can reconcile.
- **Fairness & priority:** Implement FIFO queues or weighted priorities where fairness matters, and allow admin overrides.
- **Tooling:** Provide inspection UIs, force-release endpoints, and instrumentation to spot contention before it harms SLAs.

## Interview Kit
1. **How do you avoid clock skew issues for TTLs?**  
   Base expiries on store-side time, not client clocks; rely on monotonic counters or server timestamps.
2. **What’s the role of fencing tokens?**  
   They provide a monotonically increasing identifier that downstream systems can verify, preventing stale owners from mutating shared state.
3. **How would you test lock fairness?**  
   Run stress tests with thousands of clients, collect acquisition order, and ensure starvation doesn’t occur; adjust queueing or backoff as needed.
