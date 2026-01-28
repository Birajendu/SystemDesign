# 10. NoSQL Databases and Their Trade-offs

## Problem Overview
- Match workloads to the right NoSQL engine (document, column, key-value, graph) by articulating trade-offs.

## Functional Requirements
- Provide decision matrix covering access patterns, consistency, scale, and operational maturity.
- Capture migration/interop strategies for hybrid deployments.
- Document reference architectures for at least three workload archetypes (catalog, analytics, session state).

## Non-Functional Goals
- Emphasize predictable latency (<20 ms reads for session data, <200 ms for analytics) and cost models.
- Ensure governance for schema evolution, backups, and vendor lock-in mitigation.

## Architecture Overview
- Reference stacks: Document DB + search index, Column family (Cassandra/Scylla) for time-series, Wide-key KV for sessions.
- Data ingestion pipelines handle schema-on-write vs. schema-on-read differences.

## Data Design & APIs
- Decision template fields: `Workload`, `Data shape`, `Consistency need`, `Query types`, `Scaling lever`, `Operational risks`.
- Provide CLI or spreadsheet tool to score each DB option.

## Implementation Plan
1. Inventory workloads + SLAs; fill decision templates with actual metrics.
2. Build canonical diagrams for chosen engines (partitioning, replication, failure recovery).
3. Define governance docs covering backups, schema migrations, secondary index strategy.
4. Provide SDK patterns for each engine emphasizing retries, pagination, key modeling.
5. Document migration playbooks from relational to NoSQL or vice versa.

## Testing & Validation
- Benchmark representative workloads on shortlisted engines; capture latency + throughput.
- Run failure drills (node loss, network partition) to observe consistency behavior.

## Operational Considerations
- Monitor unique metrics per engine (e.g., Cassandra repair lag, Mongo replication windows).
- Budget for cross-region replication costs and capacity planning.

## Tutorial Deep Dive
### Block Diagram
```mermaid
flowchart TD
    Workloads[[Workload Profiles]] --> Matrix[Decision Matrix & Scorecards]
    Matrix --> Engine1[Document DB\n(e.g., Mongo)]
    Matrix --> Engine2[Columnar/Time-series\n(e.g., Cassandra)]
    Matrix --> Engine3[Key-Value / Cache\n(e.g., Redis)]
    Matrix --> Engine4[Graph/Vector Stores]
    Engines[Selected Engines] --> Governance[Governance & Migration Playbooks]
```

### Design Walkthrough
- **Discovery:** Classify workloads by access pattern, consistency need, and growth; feed data into a matrix that weights each engine’s strengths/risks.
- **Reference stacks:** Document how canonical stacks look (e.g., document DB + search, wide-column + Spark) so interviews include concrete trade-offs.
- **Migration approach:** Provide patterns for dual writes, change data capture, and sunset plans when consolidating or diversifying data stores.
- **Governance:** Standardize backup, schema evolution, and observability across engines so operations remain predictable.

## Interview Kit
1. **How would you justify picking a document DB for catalog data?**  
   Highlight flexible schema, embedded documents for product variations, and indexing support; mitigate risks via schema validation and workload isolation.
2. **When is Cassandra a better fit than Mongo?**  
   For time-series or write-heavy workloads needing linear scalability and predictable latencies, provided you can model queries up front and accept limited ad-hoc querying.
3. **How do you avoid vendor lock-in?**  
   Abstract data access via service layers, keep schemas portable, and plan export/backup formats that let you migrate or operate multi-cloud when necessary.
