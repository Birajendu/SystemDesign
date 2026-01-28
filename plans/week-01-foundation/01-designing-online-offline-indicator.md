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
