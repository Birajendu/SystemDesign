# 04. Caching and Their Issues at Scale

## Problem Overview
- Maintain high cache hit rates while defending against stampedes, stale data, and replication gaps.

## Functional Requirements
- Support multiple cache classes (hot keys, streaming sessions, immutable blobs) with tailored TTL + eviction.
- Implement cache stampede protection and write policies (write-through, refresh-ahead, write-back).
- Provide multi-tier caches (edge CDN, mid-tier Redis, client caches) with invalidation APIs.

## Non-Functional Goals
- Target >95% hit rate for hot key class, rebuild cold cache within 5 minutes after node loss.
- Consistency windows configurable per data class; guarantee no stale data beyond TTL + epsilon.

## Architecture Overview
- Frontend gateway consults local L1 cache; L2 distributed cache (Redis/Memcached/DiceDB) replicates via cluster shards.
- Metadata service tags keys with class + invalidation strategy.
- Background refresh workers pre-warm caches and coalesce identical misses.

## Data Design & APIs
- Cache metadata table: `(key_pattern, class, ttl, eviction_policy, refresh_strategy)`.
- API endpoints for `POST /invalidate`, `POST /warmup`, `GET /cache-metrics`.
- Lua or eBPF scripts for token-bucket gating and deterministic jitter on TTLs.

## Implementation Plan
1. Classify datasets and map to cache tiers + TTL/eviction choices.
2. Implement read/write path wrappers with request coalescing + jittered TTL expiration.
3. Deploy background refresh workers + warmup APIs for major releases.
4. Layer observability: per-shard hit/miss, smart sampling of stale reads, and invalidation audit logs.
5. Add chaos routines: random node eviction, mass invalidation, and verifying data-plane recovery.

## Testing & Validation
- Load-test hot keys with synthetic skew to validate randomization + fallback behavior.
- Trigger partial network partition to confirm gossip-based invalidation convergence times.
- Validate fallback to source-of-truth datastore under prolonged cache outage.

## Operational Considerations
- Maintain library for safe cache clients (rate limiting, instrumentation, fallback) to avoid misuse.
- Provide dashboards for memory usage, replication lag, and invalidation volume; set alerts for stampede indicators.
