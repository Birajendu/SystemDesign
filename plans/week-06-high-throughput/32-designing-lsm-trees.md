# 32. Designing LSM Trees Ground Up

## Problem Overview
- Construct log-structured merge trees from first principles to understand write path, read path, and compaction behavior.

## Functional Requirements
- Implement WAL, memtable, immutable segments, and multi-level SSTables.
- Provide APIs for writes, reads, range scans, snapshots, and iterators.
- Include compaction strategies (tiered vs. leveled) selectable at runtime.

## Non-Functional Goals
- Ensure crash safety (WAL replay) and bounded read/write amplification (configurable thresholds).
- Support dataset growth to hundreds of GB per node with predictable latency.

## Architecture Overview
- Write path: append to WAL, update in-memory skiplist; flush to SSTable when memtable full.
- Read path: consult bloom filters, block cache; merge iterators across levels.
- Compaction scheduler merges SSTables, handles tombstones, and enforces size ratios.

## Data Design & APIs
- WAL format: `[CRC][sequence][key_length][value_length][payload]`.
- SSTable layout: data blocks, index block, bloom filter, footer with metadata.
- APIs: `Put`, `Delete`, `Get`, `Range(start,end)`, `Flush`, `Compact(level)`.

## Implementation Plan
1. Prototype WAL + memtable (skiplist/AVL) with iterators and flush triggers.
2. Build SSTable writer/reader, block cache, and bloom filters.
3. Implement leveled compaction (size-tiered -> L0-LN) and tune thresholds; add metrics (compaction bytes/sec, pending bytes).
4. Add snapshots/checkpoints by pinning memtables/SSTables and enabling consistent iterators.
5. Package CLI for benchmarking + visualizing compaction schedule.

## Testing & Validation
- Unit test WAL replay, compaction correctness, iterator ordering.
- Stress test with mixed workloads; monitor read/write amplification.
- Inject crashes mid-flush/compaction; ensure recovery.

## Operational Considerations
- Provide tuning guide for memtable size, block cache, compaction concurrency.
- Document upgrade strategy for SSTable format and migration to distributed deployments.
