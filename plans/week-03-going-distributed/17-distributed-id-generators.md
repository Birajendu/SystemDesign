# 17. Distributed ID Generators

## Problem Overview
- Generate globally unique, roughly ordered identifiers without central bottlenecks for large-scale systems.

## Functional Requirements
- Provide APIs for batch + single ID allocation with monotonic guarantees per scope.
- Encode metadata (timestamp, region, shard, sequence) for debugging and routing.
- Support backfill/migration by decoding/encoding IDs.

## Non-Functional Goals
- Sustain >1M IDs/sec with p99 latency < 2 ms per request.
- Tolerate clock skew/drift; no duplicate IDs even under failover.

## Architecture Overview
- Hybrid Snowflake design: stateless ID servers with local counters + time component.
- Coordination service assigns node IDs; health checks prevent duplicates.
- Clients optionally embed caching/batching for efficiency.

## Data Design & APIs
- ID format (64-bit): `[1 sign][41 timestamp][10 shard/node][12 sequence]` or variants.
- API: `POST /ids?count=n`, returns array + metadata. gRPC streaming for high volume.
- Monitoring of sequence exhaustion, clock rollback detection.

## Implementation Plan
1. Define ID bit layout aligning with retention + scale requirements.
2. Implement ID server that reads system clock, enforces monotonicity (waits on backward clock) and increments sequence.
3. Build registry for node IDs + health (e.g., etcd) and automate node provisioning.
4. Provide client SDKs with batching, caching, and decode helpers.
5. Add audit pipeline to detect collisions + alert if sequences near overflow.

## Testing & Validation
- Stress tests to ensure throughput; verify uniqueness via large sample.
- Simulate clock rollback (NTP issues) to confirm server waits or trips circuit breaker.
- Fail node and ensure other nodes continue issuing IDs without duplicates.

## Operational Considerations
- Monitor time sync (NTP/PTP), CPU, and sequence usage; rotate node IDs gracefully.
- Document migration/backfill steps for legacy IDs.

## Tutorial Deep Dive
### Block Diagram
```mermaid
flowchart LR
    Clients --> API[ID Generation API]
    API --> Nodes[ID Nodes\n(Snowflake Servers)]
    Nodes --> Registry[Node Registry/Coordinator]
    Nodes --> Metrics[Uniqueness & Latency Metrics]
    Registry --> Nodes
    Nodes --> Audit[ID Audit Log]
```

### Design Walkthrough
- **Server design:** Each ID node reads clock, appends node bits, and increments sequences; when clock regresses, it waits or switches to backup mode.
- **Coordination:** Node registry assigns unique node IDs, detects duplicates, and drains unhealthy nodes automatically.
- **Distribution:** Clients can request batches; SDK caches IDs to reduce latency and surfaces decode helpers for debugging.
- **Auditability:** Sample IDs into analytics to verify monotonicity and detect anomalies (collisions, gaps) quickly.

## Interview Kit
1. **How do you handle clock skew?**  
   Sync via NTP/PTP, monitor offsets, and pause issuance when skew exceeds thresholds; consider hybrid logical clocks if strict ordering is required.
2. **What if a node exhausts its sequence within a millisecond?**  
   Either block until the next millisecond tick, spin up more nodes with sharded traffic, or increase the sequence bit-width if acceptable.
3. **How would you migrate from legacy IDs?**  
   Run the new generator in shadow mode, dual-write IDs, provide translation layers, and backfill historical data asynchronously.
