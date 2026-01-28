# 01. Designing Online/Offline Indicator

## Problem Overview
- Provide near-real-time visibility of a user’s presence state across web, desktop, and mobile clients.
- Presence must reconcile multiple devices per user and survive transient network partitions.

## Functional Requirements
- Accept heartbeats via WebSocket pings, background mobile tasks, and manual overrides (e.g., DND).
- Expose a `GET /presence?user_ids=` API returning latest state + freshness timestamp.
- Stream diffs to interested parties (friends, team members) via fan-out channels.

## Non-Functional Goals
- End-to-end freshness: <2s for online transitions, <5s for offline detection.
- Support 50M concurrent connections and 500k state changes/sec with 3x burst handling.
- Preserve correctness during regional failover; data loss <0.01% of transitions.

## Architecture Overview
- Edge WebSocket gateways terminate TLS, normalize heartbeats, and publish state events to Kafka.
- Presence service consumes events, applies dedupe + precedence rules, and writes durable state to a quorum store (e.g., CockroachDB) plus Redis/DiceDB cache.
- Subscription service maintains per-user subscription lists and pushes diffs over fan-out channels throttled via token buckets.

## Data Design & APIs
- Presence table: `(user_id, device_id, state, last_seen, session_id, precedence)` with TTL cleanup jobs.
- Redis structures: `hash(user_id -> state blob)` plus sorted set keyed by `last_seen` for expiry sweeps.
- APIs include `POST /presence/override`, `WS /presence/stream`, and `GET /presence/history?user_id & window` for audit trails.

## Implementation Plan
1. Build heartbeat ingestion with protocol adapters and end-to-end tracing IDs.
2. Implement state reducer that enforces priority (manual > app > inferred) and publishes change events.
3. Layer Redis cache with write-through semantics and automatic invalidation on change.
4. Add subscription/fan-out tier with per-user diff suppression and batching.
5. Launch audit + analytics pipelines (stale ratio, false offline) prior to GA.

## Testing & Validation
- Simulate jittery clients and packet drops to validate offline thresholds.
- Run load tests on Redis + DB to verify 3x burst tolerance.
- Chaos test cross-region replication to measure RPO/RTO for presence data.

## Operational Considerations
- Alert on heartbeat backlog, Redis eviction spikes, and presence freshness p95.
- Provide runbooks for draining WebSocket gateways, revoking faulty overrides, and replaying Kafka events for rebuilds.

## Tutorial Deep Dive
### Block Diagram
```mermaid
flowchart LR
    Clients{{Web/Mobile Clients}} --> GW[Presence Gateway\n(WebSocket + REST)]
    GW --> Bus[(Event Bus/Kafka)]
    Bus --> Reducer[Presence Reducer\nConflict Resolver]
    Reducer --> Cache[(Redis/DiceDB Cache)]
    Reducer --> Store[(Durable Store\nCockroachDB/Cassandra)]
    Reducer --> Fanout[Subscription Fan-out]
    Fanout --> Clients
    Cache --> Obs[Observability + Alerting]
    Store --> Obs
```

### Design Walkthrough
- **Ingestion to normalization:** Gateways terminate sockets, normalize heartbeats, and tag them with device + freshness metadata before publishing to the bus so reducers remain stateless.
- **Conflict resolution:** Reducer enforces manual > app > inferred precedence, dedupes rapid oscillations, and emits compact change events; cache writes are idempotent so replays are safe.
- **Read fan-out:** Subscribers maintain cursors; fan-out batches diffs per user/device to keep bandwidth predictable and prevents N^2 explosions in large workspaces.
- **Durability & recovery:** Every cache mutation has matching durable write plus Kafka offset; rebuild jobs can replay offsets to hydrate cache after outages.

## Interview Kit
1. **How do you defend against oscillating network conditions causing false offline events?**  
   Use hysteresis: require multiple missed heartbeats before marking offline, combine push/pull checks, and track per-device health to avoid toggling the entire account.
2. **What’s your strategy when Redis becomes a bottleneck?**  
   Scale horizontally via consistent hashing of user IDs, shard subscription fan-out, and enable client-side caching for small cohorts; in worst cases downgrade freshness to polling while cache recovers.
3. **How would you audit presence history for compliance?**  
   Stream reducer outputs into an append-only warehouse table with TTL + GDPR tooling; build queries that reconstruct state transitions per user/device when investigations arise.
