# 24. Implementing Single-node Cache

## Problem Overview
- Build a high-performance in-memory cache running on a single node with predictable eviction and observability.

## Functional Requirements
- Provide `GET`, `SET`, `DELETE`, `INCR`, `MGET` commands with optional TTL per key.
- Support eviction policies (LRU, LFU) selectable at runtime.
- Offer snapshot + append-only log persistence for crash recovery.

## Non-Functional Goals
- Handle >1M ops/sec with sub-millisecond latency under warm cache.
- Resume from persistence with <30s startup time for 100GB data.

## Architecture Overview
- Core data structure: hash map pointing to doubly-linked lists (LRU) or count-min sketch assisted LFU.
- Event loop using epoll/libuv handles network IO; command parser optimized for pipeline.
- Persistence thread writes append log + periodic snapshot; background rehashing.

## Data Design & APIs
- Metadata per entry: `key`, `value pointer`, `ttl`, `last_access`, `freq_counter`.
- Expose RESP-compatible protocol for compatibility.
- Admin API for stats, config changes, and eviction forcing.

## Implementation Plan
1. Implement core storage engine (hash map + freelist) with lock striping.
2. Add command parser + networking (single-threaded event loop or multi-reactor) with pipelining.
3. Layer TTL wheel for expirations, background eviction + defragmentation.
4. Implement persistence (AOF + RDB snapshots) and recovery logic.
5. Instrument metrics (ops/sec, hit ratio, latency) and integrate CLI inspector.

## Testing & Validation
- Microbenchmarks for get/set throughput; compare LRU vs. LFU behavior.
- Crash/restart tests to validate persistence correctness.
- Memory fragmentation analysis via stress tests.

## Operational Considerations
- Provide scripts for resizing memory, tuning max-clients, and detecting slow commands.
- Document backup strategy and upgrade procedure for on-disk formats.
