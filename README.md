# System Design — Tutorial

This repository captures a tutorial-style design plan for every problem highlighted in the. The goal is to give you a reproducible workshop format: walk through the discovery questions, map requirements to architecture, and run fast validation loops before you ever open your IDE.

## How to use this guide
- Pick the session you intend to study, skim the **Context** to restate the problem in business terms, and write down any assumptions that differ in your environment.
- Follow the **Tutorial Steps** in order. Each step is intentionally action-oriented so you can drive a whiteboard session, facilitate cohort discussions, or solo-practice with a timer.
- Use the **Key Trade-offs** bullets as prompts to force debates. Your design should explicitly defend one side of each trade-off.
- Close with the **Hands-on Prompts** so that every session produces a tangible artifact—data model sketches, pseudo-code, dashboards, or chaos experiments.
- Each week opens with a short reminder of the engineering theme; revisit it to see how the systems tie together.

## Tutorial recipe at a glance
1. **Frame the experience** – capture primary actors, SLAs, and user journey while calling out non-goals.
2. **Shape the workload** – estimate requests per second, fan-out, storage growth, latency budget, and error budgets that will stress the system.
3. **Choose architectural primitives** – consistency knobs, data models, compute topologies, partitioning strategy, and network boundaries.
4. **Deep-dive a risky component** – pick the hottest shard, the most brittle algorithm, or the component with the most unknowns.
5. **Operationalize** – monitoring, playbooks, chaos drills, security controls, and incremental delivery plan.

---

## Week 1 – Foundation
Primer: build intuition around latency budgets, synchronous vs. asynchronous flows, and communication primitives before optimizing any code.

#### 1. Designing Online/Offline Indicator — [Detailed Plan](plans/week-01-foundation/01-designing-online-offline-indicator.md)
- **Context** – Track user presence with second-level accuracy for chat or collaboration apps without flooding infrastructure.
- **Tutorial Steps**
  1. Bucket presence sources (web sockets, mobile heartbeats, manual overrides) and map freshness requirements per client.
  2. Model a write-heavy, read-even cache (e.g., Redis or DiceDB) with TTLs, and design fallback reads from durable storage when cache is cold.
  3. Simulate presence fan-out to friends/devices and cap broadcast rate using diff-based updates.
  4. Instrument health metrics (stale ratio, heartbeat failure rate) and outline alert thresholds.
- **Key Trade-offs** – Push vs. pull updates; cache-only vs. cache + source of truth; last-write-wins vs. vector clocks for multi-device presence.
- **Hands-on Prompts** – Code a heartbeat daemon with jitter, inject packet loss, and observe how quickly presence reconverges; sketch Grafana panels that prove latency SLAs.

#### 2. 3 Mental Models and Building the Right Intuition — [Detailed Plan](plans/week-01-foundation/02-mental-models-and-intuition.md)
- **Context** – Codify reusable heuristics (e.g., back-pressure triangles, CAP toggles, hot-path decomposition) for future sessions.
- **Tutorial Steps**
  1. Write down three canonical tensions (latency vs. throughput, consistency vs. availability, build vs. buy) and relate them to recent incidents.
  2. For each, draw a quadrant diagram and place at least two systems from your experience to show why the trade-off was chosen.
  3. Build “5 why” questionnaires so every future design review starts with probing user impact before tech choices.
  4. Define a rubric for when to zoom into detail (P0 risk, new workload, regulatory constraints).
- **Key Trade-offs** – Heuristics vs. formal proofs; investing in modeling vs. shipping experiments; intuition from data vs. anecdotes.
- **Hands-on Prompts** – Facilitate a 30-minute retro of a past launch using these models; document one intuition failure and how to correct for it.

#### 3. Practical Nuances Around Connection Pools and DB Proxies — [Detailed Plan](plans/week-01-foundation/03-connection-pools-and-db-proxies.md)
- **Context** – Prevent thundering herds and saturating databases by sizing pools and proxies intelligently.
- **Tutorial Steps**
  1. Quantify concurrency needs per service (qps × avg latency) and derive connection pool upper bounds via Little’s Law.
  2. Design a proxy tier (pgBouncer, Envoy) with transaction vs. session pooling and failure isolation per tenant.
  3. Add back-pressure: circuit breakers, queue limits, adaptive throttling tied to DB health signals.
  4. Plan observability—pool utilization histograms, wait time traces, and leak detection.
- **Key Trade-offs** – Multiplexing vs. dedicated connections; pool size vs. transaction latency; client-side retries vs. server-side queuing.
- **Hands-on Prompts** – Replay load tests with gradual pool shrinkage; script chaos where proxy nodes reboot during peak throughput.

#### 4. Caching and Their Issues at Scale — [Detailed Plan](plans/week-01-foundation/04-caching-at-scale.md)
- **Context** – Keep cache hit rate high while defending against stampedes, replication lag, and invalidation complexity.
- **Tutorial Steps**
  1. Segment data (hot keys, streaming data, immutable blobs) and assign eviction and TTL policies per class.
  2. Decide write strategies (write-through, write-back, refresh-ahead) and codify fallback to persistent storage.
  3. Address consistency: token buckets for cache stampede protection, quorum reads for correctness, gossip for invalidation.
  4. Build runbooks for cold-starts, node replacements, and region evacuation.
- **Key Trade-offs** – Cache correctness vs. freshness, latency vs. consistency, memory footprint vs. replication.
- **Hands-on Prompts** – Implement request coalescing middleware; simulate cache node loss and measure rebuild window.

#### 5. Async Processing and Kafka Essentials — [Detailed Plan](plans/week-01-foundation/05-async-processing-and-kafka.md)
- **Context** – Decouple writes from heavy processing with log-based queues and idempotent consumers.
- **Tutorial Steps**
  1. Capture producer throughput, ordering guarantees, and message durability SLAs.
  2. Partition topics by entity ID, define compaction vs. retention windows, and tune segment sizes.
  3. Draw consumer groups (exactly-once, at-least-once) and define idempotency keys and retry policies.
  4. Document disaster recovery (mirrormaker, tiered storage) and schema governance.
- **Key Trade-offs** – Throughput vs. ordering; synchronous writes vs. async ack; compaction vs. retention.
- **Hands-on Prompts** – Build a replay script that rehydrates downstream state from Kafka; create a consumer lag alert that includes remediation steps.

#### 6. Communication Paradigms and Deployment Log Streamer — [Detailed Plan](plans/week-01-foundation/06-communication-paradigms-and-log-streamer.md)
- **Context** – Choose between REST, gRPC, WebSocket, and pub/sub while building a log streamer for deployments.
- **Tutorial Steps**
  1. Classify interactions (request/response, streaming, fire-and-forget) and match them to protocols.
  2. Design a log ingestion plane that tails deployment logs, batches them, and streams incremental updates with pagination.
  3. Secure channels with mutual TLS, tenant isolation, and redact secrets before broadcasting.
  4. Provide fallbacks—persistent cursor checkpoints, offline downloads, and retention policies.
- **Key Trade-offs** – Text vs. binary protocols; push vs. poll updates; simplicity vs. feature richness.
- **Hands-on Prompts** – Prototype a WebSocket gateway with exponential back-off reconnects; capture stress metrics for 10k concurrent viewers.

---

## Week 2 – Databases Deep Dive
Primer: understand data models, consistency models, and how operational decisions impact latency and cost when data sets grow.

#### 7. Pessimistic Locking on Relational DBs — [Detailed Plan](plans/week-02-databases/07-pessimistic-locking.md)
- **Context** – Guarantee serialization for high-value transactions while minimizing lock contention.
- **Tutorial Steps**
  1. Classify workloads that merit SELECT … FOR UPDATE vs. application-level locks.
  2. Define lock granularity (row, page, table) given typical query shapes.
  3. Model timeout policies, deadlock detection, and retry envelopes.
  4. Instrument lock wait times and surface them via tracing.
- **Key Trade-offs** – Strict correctness vs. throughput; large vs. fine-grained locks; database locks vs. distributed locks.
- **Hands-on Prompts** – Replay conflicting transactions in a sandbox DB and document the wait graph.

#### 8. Designing a Scalable SQL-backed KV Store — [Detailed Plan](plans/week-02-databases/08-sql-backed-kv-store.md)
- **Context** – Offer key-value semantics without abandoning relational guarantees.
- **Tutorial Steps**
  1. Choose sharding keys and map them to table partitions or PostgreSQL Citus shards.
  2. Define schema (key, version, payload, ttl) and indexing strategy.
  3. Plan read/write paths: prepared statements, batched writes, and read replicas for fan-out.
  4. Outline background jobs for TTL eviction, compaction, and secondary index rebuilds.
- **Key Trade-offs** – Normalized schema vs. JSONB blobs; auto-increment vs. client-generated IDs; scale-up vs. federation.
- **Hands-on Prompts** – Benchmark point reads vs. range scans; simulate failover to replica nodes.

#### 9. How to Scale Relational Databases — [Detailed Plan](plans/week-02-databases/09-scaling-relational-databases.md)
- **Context** – Keep a single logical database healthy under explosive growth.
- **Tutorial Steps**
  1. Baseline current workload (OLTP vs. OLAP) and identify the first saturation point (CPU, IO, connections).
  2. Layer scale-up levers (indexes, query plans, connection pooling) before cross-shard moves.
  3. Design a read-heavy replica strategy plus write splitting per domain.
  4. Plan vertical partitioning, sharding, and eventual multi-region replication.
- **Key Trade-offs** – Strong vs. eventual consistency; synchronous vs. async replication; operational simplicity vs. cost.
- **Hands-on Prompts** – Create a playbook for online schema changes; measure replication lag under burst writes.

#### 10. NoSQL Databases and Their Trade-offs — [Detailed Plan](plans/week-02-databases/10-nosql-tradeoffs.md)
- **Context** – Match workloads to document, columnar, key-value, or graph data stores.
- **Tutorial Steps**
  1. Compare access patterns (catalog, time-series, session state) and align them to storage engines.
  2. Define consistency and durability requirements per workload.
  3. Evaluate indexing strategies, partitioning schemes, and query limitations.
  4. Document migration paths or hybrid patterns (polyglot persistence).
- **Key Trade-offs** – Schema flexibility vs. transactional guarantees; secondary indexes vs. write amplification; vendor lock-in vs. DIY.
- **Hands-on Prompts** – Write a decision matrix that maps three workloads to different NoSQL engines.

#### 11. When to Use and Not to Use Graph Databases — [Detailed Plan](plans/week-02-databases/11-graph-database-usage.md)
- **Context** – Decide whether relationships warrant native graph storage versus relational joins or document embeddings.
- **Tutorial Steps**
  1. Quantify graph density, pattern query types, and traversal depth.
  2. Compare storage options: property graph, adjacency lists, column families.
  3. Assess latency/throughput requirements for traversals and update frequency.
  4. Plan data lifecycle—bulk loads, incremental sync, cache layers for hot subgraphs.
- **Key Trade-offs** – Query expressiveness vs. operational complexity; ACID vs. eventual; specialized tooling vs. ecosystem support.
- **Hands-on Prompts** – Implement the same query in Neo4j and SQL to highlight performance gaps.

#### 12. Designing Slack's Realtime Text Communication — [Detailed Plan](plans/week-02-databases/12-slack-realtime-text.md)
- **Context** – Multi-tenant chat with channel fan-out, typing indicators, and read receipts.
- **Tutorial Steps**
  1. Map user-tenant-channel hierarchy and throughput targets.
  2. Design gateway tier (HTTP upgrade/WebSockets), identify fan-out strategy (publish-subscribe vs. persistent connections).
  3. Choose storage: append-only logs per channel plus search indexes; define consistency for edits/deletes.
  4. Outline delivery guarantees, ack paths, and offline sync flows.
- **Key Trade-offs** – Single global WebSocket vs. per-shard; acked delivery vs. fire-and-forget; data localization vs. co-mingled tenants.
- **Hands-on Prompts** – Simulate 1k-member channel bursts and monitor message latency percentiles.

#### 13. Scaling WebSockets — [Detailed Plan](plans/week-02-databases/13-scaling-websockets.md)
- **Context** – Maintain millions of concurrent connections with predictable resource usage.
- **Tutorial Steps**
  1. Forecast concurrent sessions, handshake rate, and sustained throughput.
  2. Architect gateway pools with consistent hashing of connection IDs and sticky routing behind L4 load balancers.
  3. Plan graceful degradation: connection draining, ping/pong health, fallback to long-polling.
  4. Harden operations—capacity autoscaling, TLS session resumption, and quota enforcement.
- **Key Trade-offs** – Stateful vs. stateless gateways; TLS termination edge vs. origin; vertical vs. horizontal scaling.
- **Hands-on Prompts** – Load test with h2load/websocat, gradually kill nodes, and observe reconnection storms.

---

## Week 3 – Going Distributed
Primer: face coordination problems head-on—leadership elections, lock orchestration, and partition management.

#### 14. Designing Load Balancers Without a Single Point of Failure — [Detailed Plan](plans/week-03-going-distributed/14-load-balancers-no-spof.md)
- **Context** – Provide horizontal scaling and resilience for L4/L7 traffic routing.
- **Tutorial Steps**
  1. Map client entry points (Anycast, DNS, Global Accelerator) and failover policies.
  2. Split control plane (config, health) from data plane (packet forwarding) and outline redundancy per layer.
  3. Define consistent hashing or least-connections routing plus session affinity rules.
  4. Document observability: per hop latency, drop counters, synthetic checks.
- **Key Trade-offs** – Software vs. hardware load balancers; centralized config vs. distributed; stateful vs. stateless routing.
- **Hands-on Prompts** – Prototype with Envoy or HAProxy clusters; run chaos to ensure zero SPOF.

#### 15. Leader Election and Self-healing Distributed Systems — [Detailed Plan](plans/week-03-going-distributed/15-leader-election.md)
- **Context** – Coordinate clusters so only one node mutates state while others standby.
- **Tutorial Steps**
  1. Choose coordination primitive (Raft, ZooKeeper, etcd) and heartbeat timing.
  2. Define election timeouts relative to network latency and clock drift.
  3. Plan state replication, fencing tokens, and idempotent transitions.
  4. Enumerate failure scenarios (split brain, slow follower) and mitigation scripts.
- **Key Trade-offs** – Short vs. long election timeouts; consensus complexity vs. operational simplicity; synchronous vs. async replication.
- **Hands-on Prompts** – Implement a simple Raft cluster and visualize leader transitions under packet delay.

#### 16. Implementing Remote and Distributed Locks — [Detailed Plan](plans/week-03-going-distributed/16-distributed-locks.md)
- **Context** – Provide cross-service critical sections without resorting to DB locks.
- **Tutorial Steps**
  1. Select locking semantics (mutual exclusion, read/write locks, leases) and TTLs.
  2. Design lock storage (Redis SET NX, DynamoDB conditional writes, ZooKeeper ephemeral nodes).
  3. Handle failures: lock renewal, fencing tokens, distributed fairness.
  4. Provide client libraries with instrumentation for contention.
- **Key Trade-offs** – Lease duration vs. safety; centralized vs. decentralized coordination; fairness vs. throughput.
- **Hands-on Prompts** – Stress test lock acquisition with thousands of contenders.

#### 17. Distributed ID Generators — [Detailed Plan](plans/week-03-going-distributed/17-distributed-id-generators.md)
- **Context** – Provide unique, ordered identifiers without central bottlenecks.
- **Tutorial Steps**
  1. List requirements: monotonicity, locality, sharding hints, and lifespan.
  2. Compare snowflake-style, database sequences with caching, and random UUID variants.
  3. Design clock sync and rollback handling for time-based schemes.
  4. Plan migration/backfill strategy and how clients decode IDs.
- **Key Trade-offs** – Global ordering vs. availability; human readability vs. entropy; strong time sync vs. resilience.
- **Hands-on Prompts** – Implement a snowflake generator and validate uniqueness across regions.

#### 18. Handling Hot-Shard Problems — [Detailed Plan](plans/week-03-going-distributed/18-handling-hot-shards.md)
- **Context** – Avoid partitions receiving disproportionate traffic causing latency spikes.
- **Tutorial Steps**
  1. Instrument shard metrics (qps, p99 latency, CPU) and build hot-spot detection thresholds.
  2. Plan mitigation options: key randomization, adaptive sharding, cache overlays.
  3. Automate rebalancing workflows (split, move, or duplicate shards) with minimal downtime.
  4. Document SLA downgrade plans when mitigation lags demand.
- **Key Trade-offs** – Deterministic sharding vs. randomness; immediate rebalancing vs. background reshuffle; hardware overprovisioning vs. algorithmic solutions.
- **Hands-on Prompts** – Replay a traffic trace with skewed distribution and evaluate each mitigation.

---

## Week 4 – Building Social Networks
Primer: design consumer-facing features with privacy controls, high fan-out, and creative media pipelines.

#### 19. Designing and Implementing Photos Upload at Scale — [Detailed Plan](plans/week-04-social-networks/19-photos-upload-at-scale.md)
- **Context** – Accept uploads worldwide, support resumable transfers, and back them with durable storage plus CDNs.
- **Tutorial Steps**
  1. Map client upload flows (chunked HTTP, direct-to-S3) and security (signed URLs, malware scanning).
  2. Design ingestion service that validates metadata, triggers background resizing, and writes to multi-region buckets.
  3. Build asynchronous processing (thumbnails, EXIF filters) and CDN cache pre-warming.
  4. Document lifecycle policies and deletion workflows.
- **Key Trade-offs** – Direct-to-cloud vs. proxy uploads; synchronous vs. async image processing; cost vs. redundancy.
- **Hands-on Prompts** – Prototype multipart upload with deliberate packet loss; profile thumbnail pipeline latency.

#### 20. Implementing Private Photos for Instagram — [Detailed Plan](plans/week-04-social-networks/20-private-photos-instagram.md)
- **Context** – Enforce ACLs, ephemeral sharing, and access auditing for private media.
- **Tutorial Steps**
  1. Define privacy models (close friends, followers, custom lists) and encode them into policy stores.
  2. Integrate signed URLs with short TTLs plus on-the-fly authorization checks.
  3. Plan notification and caching strategy respecting ACLs (per-user cache, shared-nothing CDNs, encrypted blobs).
  4. Audit every access with immutable logs and retention policies.
- **Key Trade-offs** – Performance vs. fine-grained ACL checks; encryption at rest vs. key management overhead; caching vs. privacy leaks.
- **Hands-on Prompts** – Design threat model and run access replay to validate audit completeness.

#### 21. Designing Gravatar and Dynamic OG Images — [Detailed Plan](plans/week-04-social-networks/21-gravatar-og-images.md)
- **Context** – Generate avatars or preview images on-demand with caching and personalization.
- **Tutorial Steps**
  1. Model input variants (hash-based avatar lookup, query parameters) and rate-limits.
  2. Build template rendering service (SVG/HTML-to-image) with caching layers (edge CDN + mid-tier cache).
  3. Plan storage for user-uploaded assets with hashing to deduplicate.
  4. Outline fallback/resiliency when rendering fails.
- **Key Trade-offs** – Pre-generation vs. just-in-time rendering; CDN cache TTL vs. freshness; personalization vs. cache hit rate.
- **Hands-on Prompts** – Implement signed rendering URLs; monitor CPU for templating under burst traffic.

#### 22. Designing Concurrent Hashtag Counter — [Detailed Plan](plans/week-04-social-networks/22-concurrent-hashtag-counter.md)
- **Context** – Track trending hashtags without losing accuracy under concurrent writes.
- **Tutorial Steps**
  1. Decide counting model: sharded counters, CRDTs, or streaming aggregations.
  2. Partition counters by hashtag/time bucket and define merge cadence.
  3. Provide query APIs for trending windows (1m, 1h, 1d) with ranking.
  4. Set up anti-abuse checks and dedupe logic.
- **Key Trade-offs** – Accuracy vs. latency; centralized counters vs. distributed; memory heavy vs. recomputation.
- **Hands-on Prompts** – Build sharded counter demo using Redis + Lua and compare to Kafka Streams implementation.

#### 23. Designing Message Indicators — [Detailed Plan](plans/week-04-social-networks/23-message-indicators.md)
- **Context** – Build seen/delivered/typing indicators with minimal bandwidth.
- **Tutorial Steps**
  1. Map indicator states and priority (typing < sent < delivered < seen).
  2. Choose encoding (bitfields) and transport (piggyback on existing WebSocket events).
  3. Persist receipts with TTL for compliance and dispute resolution.
  4. Define throttling to avoid spamming clients.
- **Key Trade-offs** – Accuracy vs. privacy (read receipts opt-out); server compute vs. client compute; storage cost vs. retention.
- **Hands-on Prompts** – Emulate slow clients and confirm indicator ordering remains consistent.

---

## Week 5 – Building Storages
Primer: explore cache-heavy workloads, migrations, and log-structured data engines.

#### 24. Implementing Single-node Cache — [Detailed Plan](plans/week-05-storages/24-single-node-cache.md)
- **Context** – Build a performant in-memory cache with predictable eviction.
- **Tutorial Steps**
  1. Decide datastructures (hash map + linked list) and eviction policy (LRU, LFU).
  2. Support TTLs, serialization formats, and metrics counters.
  3. Integrate persistence (snapshotting, append log) for crash recovery.
  4. Provide administrative APIs for introspection.
- **Key Trade-offs** – Simplicity vs. features (transactions, clustering); heap allocation vs. slab; TTL accuracy vs. CPU cost.
- **Hands-on Prompts** – Profile GC impact; build flame graph before/after optimizing locks.

#### 25. DiceDB Internals and Maximizing Single-node Performance — [Detailed Plan](plans/week-05-storages/25-dicedb-single-node-performance.md)
- **Context** – Tune a Redis-compatible cache (DiceDB) for throughput.
- **Tutorial Steps**
  1. Identify bottlenecks (syscalls, network stack, command parser).
  2. Enable I/O uring or epoll loops, pipelining, and multi-threaded networking.
  3. Optimize memory allocator, data structures, and compression.
  4. Set up benchmarking harness with latency histograms.
- **Key Trade-offs** – Latency vs. throughput; custom protocol tweaks vs. client compatibility; memory footprint vs. CPU.
- **Hands-on Prompts** – Run redis-benchmark with varied pipeline depths and capture perf counters.

#### 26. Operational Complexities of Consistent Hashing — [Detailed Plan](plans/week-05-storages/26-consistent-hashing-operations.md)
- **Context** – Keep shard assignments stable while nodes churn.
- **Tutorial Steps**
  1. Implement ring construction with virtual nodes and weight distribution.
  2. Plan node addition/removal workflows plus data migration volumes.
  3. Monitor imbalance metrics and automate ring rebalancing.
  4. Document fallbacks when metadata store is unavailable.
- **Key Trade-offs** – Rebalance complexity vs. uniform distribution; centralized ring management vs. client libraries; consistent hashing vs. rendezvous.
- **Hands-on Prompts** – Visualize key movement when adding nodes; create regression tests for hashing libraries.

#### 27. No Downtime Data Migration — [Detailed Plan](plans/week-05-storages/27-no-downtime-data-migration.md)
- **Context** – Move datasets or schema without user-visible outages.
- **Tutorial Steps**
  1. Decide migration style (dual writes, change data capture, blue/green).
  2. Plan cutover sequencing, validation queries, and abort criteria.
  3. Instrument shadow traffic to detect divergences.
  4. Prepare rollback procedures and communication plan.
- **Key Trade-offs** – Speed vs. safety; synchronous cutover vs. gradual; tooling complexity vs. manual scripts.
- **Hands-on Prompts** – Dry-run a migration in staging, capture diff metrics, and document learnings.

#### 28. Designing a Word Dictionary Without a DB — [Detailed Plan](plans/week-05-storages/28-word-dictionary-without-db.md)
- **Context** – Serve word lookups with prefix queries using file or memory structures.
- **Tutorial Steps**
  1. Choose storage (DAWG, trie, sorted files) and compression techniques.
  2. Design lookup API supporting prefix, suffix, and fuzzy matches.
  3. Plan update workflow (batch rebuild vs. incremental) and deployment packaging.
  4. Add caching and offline distribution strategy.
- **Key Trade-offs** – Memory vs. disk usage; rebuild complexity vs. incremental writes; search flexibility vs. latency.
- **Hands-on Prompts** – Implement double-array trie prototype and benchmark memory footprint.

#### 29. Designing Log-Structured KV Store — [Detailed Plan](plans/week-05-storages/29-log-structured-kv-store.md)
- **Context** – Build LSM-based storage with compaction and bloom filters.
- **Tutorial Steps**
  1. Define SSTable layout, memtable flushing policy, and WAL durability.
  2. Choose compaction strategy (leveled vs. tiered) and tune thresholds.
  3. Integrate bloom filters and indexes for fast reads.
  4. Create maintenance plans for repair, backup, and recovery.
- **Key Trade-offs** – Write amplification vs. read amplification; compaction pause vs. background resources; replication vs. recovery speed.
- **Hands-on Prompts** – Simulate mixed workload and visualize compaction cascades.

---

## Week 6 – Building High Throughput Systems
Primer: optimize for concurrency, streaming media, and privacy in data-intensive flows.

#### 30. Designing S3 — [Detailed Plan](plans/week-06-high-throughput/30-designing-s3.md)
- **Context** – Object storage with 11 nines durability, geo-redundancy, and simple APIs.
- **Tutorial Steps**
  1. Break down namespace (buckets, objects) and request mix (PUT, GET, LIST, multipart uploads).
  2. Plan control plane (metadata, auth) vs. data plane (storage nodes) separation.
  3. Design replication pipeline, erasure coding, and consistency model (read-after-write for new objects, eventual for overwrite).
  4. Define billing, lifecycle rules, and security controls (IAM, encryption).
- **Key Trade-offs** – Strong vs. eventual consistency; replication factor vs. erasure coding; latency vs. durability.
- **Hands-on Prompts** – Prototype a simplified object store with metadata DB + blob store; run chaos to delete metadata and measure blast radius.

#### 31. Designing Multi-tiered Orders for Amazon — [Detailed Plan](plans/week-06-high-throughput/31-multi-tier-orders-amazon.md)
- **Context** – Handle shopping cart, payment, inventory, and fulfillment with multiple services.
- **Tutorial Steps**
  1. Draw domain boundaries (cart, pricing, payments, logistics) and event flows.
  2. Define order state machine with idempotent transitions and compensation actions.
  3. Plan data ownership per tier and cross-service contracts (events, APIs).
  4. Build resilience via sagas, retries, and fallback experiences.
- **Key Trade-offs** – Consistency vs. availability; synchronous orchestration vs. choreography; shared DB vs. per-service DB.
- **Hands-on Prompts** – Write a saga sequence with failure injections for payment decline and inventory shortage.

#### 32. Designing LSM Trees Ground Up — [Detailed Plan](plans/week-06-high-throughput/32-designing-lsm-trees.md)
- **Context** – Understand memtables, SSTables, and compaction for write-heavy DBs.
- **Tutorial Steps**
  1. Start with write path (WAL + memtable) and flush triggers.
  2. Outline read path with bloom filters, block cache, and range scans.
  3. Compare leveled vs. tiered compaction and choose when to switch.
  4. Add features like snapshots, TTL, and bulk loads.
- **Key Trade-offs** – Write amplification vs. space amplification; compaction frequency vs. tail latency; sequential vs. random IO.
- **Hands-on Prompts** – Implement a toy LSM in Go/Python and analyze compaction logs.

#### 33. Ads in Live Streaming — [Detailed Plan](plans/week-06-high-throughput/33-ads-in-live-streaming.md)
- **Context** – Inject targeted ads into live video with minimal buffering.
- **Tutorial Steps**
  1. Model streaming pipeline (ingest, transcode, packager, CDN) and ad decision points.
  2. Design SSAI (server-side ad insertion) or CSAI, with timeline stitching and DRM.
  3. Define personalization inputs, caching, and measurement (beacons, fraud detection).
  4. Set SLOs for latency and fallback (slates) if ad fails.
- **Key Trade-offs** – Server-side vs. client-side insertion; personalization depth vs. latency; monetization vs. QoE.
- **Hands-on Prompts** – Build manifest manipulation prototype and monitor rebuffer time.

#### 34. Privacy and High Availability in Live Streaming — [Detailed Plan](plans/week-06-high-throughput/34-privacy-and-ha-live-streaming.md)
- **Context** – Protect streams (encrypted, access-controlled) while maintaining uptime.
- **Tutorial Steps**
  1. Layer DRM, tokenized URLs, and geo-fencing policies.
  2. Build redundancy (multi-region ingest, hot-hot packagers, CDN multi-homing).
  3. Instrument QoE metrics (startup time, rebuffer ratio) tied to privacy controls.
  4. Plan incident workflows for key leaks or region outages.
- **Key Trade-offs** – Security rigor vs. latency; redundancy cost vs. availability; viewer privacy vs. analytics depth.
- **Hands-on Prompts** – Rotate encryption keys mid-stream; failover ingest nodes while measuring QoE.

---

## Week 7 – IR Systems and Adhoc Designs
Primer: design retrieval-heavy systems and time-sensitive commerce flows.

#### 35. Designing Recent Searches — [Detailed Plan](plans/week-07-ir-adhoc/35-recent-searches.md)
- **Context** – Show per-user search history with privacy and storage limits.
- **Tutorial Steps**
  1. Define retention policy, deduplication, and personalization signals.
  2. Choose storage (Redis, Cassandra) with per-user partitioning.
  3. Implement ranking (recency, frequency) and suggestion APIs.
  4. Offer privacy controls (clear history) and compliance logging.
- **Key Trade-offs** – Client vs. server storage; personalization vs. privacy; sync writes vs. eventual.
- **Hands-on Prompts** – Prototype TTL-based history service; evaluate encryption overhead for PII.

#### 36. Designing Cricbuzz's Text Commentary — [Detailed Plan](plans/week-07-ir-adhoc/36-cricbuzz-text-commentary.md)
- **Context** – Stream ball-by-ball updates with low latency.
- **Tutorial Steps**
  1. Model editorial workflows, message schema, and translation/localization needs.
  2. Build ingestion tools, moderation, and multi-language pipelines.
  3. Choose delivery: SSE/WebSocket + caching for watchers.
  4. Track consistency between scoreboard, commentary, and push notifications.
- **Key Trade-offs** – Manual vs. automated inputs; fan-out latency vs. bandwidth; caching vs. personalization.
- **Hands-on Prompts** – Simulate load for a popular match; test fallback to SMS/push when WebSocket drops.

#### 37. Instagram Live Reactions — [Detailed Plan](plans/week-07-ir-adhoc/37-instagram-live-reactions.md)
- **Context** – Capture billions of emoji taps without overwhelming stream.
- **Tutorial Steps**
  1. Batch client reactions, send aggregated deltas.
  2. Use regional in-memory aggregators with CRDTs or sharded counters.
  3. Overlay results in the video stream with smoothing and anti-spam.
  4. Persist aggregated stats for analytics.
- **Key Trade-offs** – Real-time accuracy vs. smoothing; client CPU vs. server CPU; storing raw vs. aggregated events.
- **Hands-on Prompts** – Build aggregator that flushes every 250 ms; measure jitter tolerance for overlay animation.

#### 38. Designing Distributed Task Scheduler — [Detailed Plan](plans/week-07-ir-adhoc/38-distributed-task-scheduler.md)
- **Context** – Execute scheduled jobs reliably across clusters.
- **Tutorial Steps**
  1. Define job metadata (cron, retries, idempotency).
  2. Choose coordination (leader/follower, consistent hashing) for job ownership.
  3. Implement worker heartbeats, leasing, and failure recovery.
  4. Provide APIs for pause/resume, rate limits, and observability.
- **Key Trade-offs** – Central scheduler vs. decentralized; scheduling accuracy vs. throughput; persistence store vs. in-memory.
- **Hands-on Prompts** – Stress test with skewed cron schedules; document how to recover orphaned jobs.

#### 39. Designing and Implementing Flash Sale — [Detailed Plan](plans/week-07-ir-adhoc/39-flash-sale.md)
- **Context** – Handle massive traffic spikes with inventory guarantees.
- **Tutorial Steps**
  1. Prepare pre-registration, captcha, and queueing strategies to smooth load.
  2. Keep inventory in an in-memory, transactional store with optimistic locking or reservation tokens.
  3. Design degraded UX (waiting rooms, progressive rollout) and live telemetry dashboards.
  4. Automate kill switches and rollback plans.
- **Key Trade-offs** – Fairness vs. resource usage; strict ordering vs. batching; complexity vs. speed to market.
- **Hands-on Prompts** – Replay synthetic spike tests; run chaos by injecting double-spend attempts.

---

## Week 8 – Building Algorithmic Systems
Primer: lean on clever algorithms to make synchronization and personalization feasible at scale.

#### 40. Approach Behind GitHub's File Sync — [Detailed Plan](plans/week-08-algorithmic/40-github-file-sync.md)
- **Context** – Sync repositories efficiently with delta encoding and conflict resolution.
- **Tutorial Steps**
  1. Break files into chunks, hash them, and design delta transfer logic.
  2. Handle concurrency: optimistic merge, conflict detection, and resumable uploads.
  3. Plan metadata store for device states, version vectors, and permissions.
  4. Design offline-first workflow with background reconciliation.
- **Key Trade-offs** – Chunk size vs. dedup efficiency; central coordination vs. peer-to-peer; bandwidth vs. CPU.
- **Hands-on Prompts** – Implement rolling hash diff prototype; simulate concurrent edits on flaky networks.

#### 41. Understanding and Scaling GeoSpatial Search — [Detailed Plan](plans/week-08-algorithmic/41-geospatial-search.md)
- **Context** – Run proximity queries with fast response time.
- **Tutorial Steps**
  1. Select indexing method (R-tree, GeoHash, S2) aligned with region density.
  2. Design ingestion for updates (moving objects) and TTL for stale data.
  3. Build query planner supporting bounding boxes, radius, and ranking by distance or score.
  4. Address multi-region data replication and compliance (geo-fencing).
- **Key Trade-offs** – Precision vs. storage; precomputed tiles vs. on-the-fly calculations; CPU vs. memory for indexes.
- **Hands-on Prompts** – Compare GeoHash precision levels on a heatmap; profile query execution for dense metros vs. sparse areas.

#### 42. Designing and Implementing User Affinity Service — [Detailed Plan](plans/week-08-algorithmic/42-user-affinity-service.md)
- **Context** – Recommend connections/items based on behavioral signals.
- **Tutorial Steps**
  1. Define feature sets (co-engagement, co-location, follow graph) and freshness requirements.
  2. Select storage for embeddings/scores (vector DB, Redis) with batch + real-time updates.
  3. Design scoring API, AB-testing hooks, and explanation logging.
  4. Close the loop with feedback ingestion (clicks, hides) and model retraining cadence.
- **Key Trade-offs** – Real-time personalization vs. offline batch; explainability vs. model complexity; latency vs. recall.
- **Hands-on Prompts** – Build a candidate-generation job plus online reranker stub; validate fairness metrics across cohorts.

---

Happy designing! Pair these plans with the official cohort recordings and prototype each system to cement the concepts.
