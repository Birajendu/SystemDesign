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

## Tutorial Deep Dive
### Block Diagram
```mermaid
flowchart LR
    Clients --> Protocol[RESP/TCP Interface]
    Protocol --> Engine[Cache Engine\n(Hash Map + LRU/LFU)]
    Engine --> TTLWheel[TTL Wheel / Expirer]
    Engine --> Persistence[(AOF/Snapshot Storage)]
    Engine --> Metrics[Metrics & Debug Console]
    Persistence --> Recovery[Recovery Flow]
```

### Design Walkthrough
- **Core engine:** Use hash map + linked list for O(1) access, apply lock striping or single-threaded event loop for determinism.
- **Eviction & TTL:** Maintain frequency counters or LRU list, run TTL wheel to batch expirations, and expose stats so clients can tune TTLs.
- **Networking:** Implement RESP parser, pipelining, and optional clustering hooks even on single node to prepare for scaling.
- **Durability:** Append-only file plus periodic snapshot ensures crash recovery with bounded data loss; provide tools to replay logs.

## Interview Kit
1. **Why choose LFU over LRU?**  
   LFU favors frequently accessed keys even if they were accessed earlier; choose when workloads exhibit periodic bursts and you can tolerate tracking overhead.
2. **How do you minimize GC/fragmentation?**  
   Use custom allocators/slabs, reuse buffers, and schedule defragmentation when CPU is idle; monitor fragmentation metrics.
3. **What’s the recovery story after crash?**  
   Replay AOF to rebuild latest state, apply snapshot for fast bootstrap, and verify checksum/version compatibility before accepting traffic.
