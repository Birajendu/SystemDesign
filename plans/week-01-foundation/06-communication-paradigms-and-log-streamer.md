# 06. Communication Paradigms & Deployment Log Streamer

## Problem Overview
- Build a deployment log streamer that supports live tails, historical pagination, and tenant isolation using the right protocol mix.

## Functional Requirements
- Ingest logs from CI/CD agents via gRPC/HTTPS with auth tokens.
- Offer live streaming (WebSocket/SSE) and historical queries via REST with pagination + filters.
- Support bookmarking/cursors, redact secrets, and provide download/export options.

## Non-Functional Goals
- Live stream latency < 1s from agent to client at 99p, backlog retention 30 days.
- Handle 10k concurrent viewers per deployment with auto-scaling fan-out tiers.
- Comply with SOC2 logging (immutability, audit trails).

## Architecture Overview
- Agents push logs to ingestion service (gRPC) -> Kafka topic -> log store (ClickHouse/Elastic/S3 + index).
- Streamer service reads from Kafka for live tail, persists offsets per client, and exposes WebSocket endpoints.
- REST API queries ClickHouse/S3 for historical data with caching + chunked responses.

## Data Design & APIs
- Log schema: `(deployment_id, step_id, timestamp, level, message, metadata, redaction_state)`.
- APIs: `POST /logs`, `WS /deployments/{id}/stream`, `GET /deployments/{id}/logs?cursor&till`.
- Cursor format encodes `(offset, wallclock)` for resuming streams.

## Implementation Plan
1. Define ingestion protobuf schema + auth (JWT, workload identity) and ship agent libraries.
2. Stand up Kafka topic + consumer that writes to columnar store with TTL policies.
3. Implement streaming gateway with backpressure, multiplexing, and heartbeat-based liveness.
4. Build historical query service with index filters, gzip chunking, and CDN caching for static downloads.
5. Add governance: redaction pipeline, access logs, retention enforcement, and tenant quotas.

## Testing & Validation
- Replay large deployments (multi-GB) to ensure streaming does not starve UI.
- Pen-test log redaction to avoid secret leaks.
- Run chaos experiments dropping connections to confirm resumable cursors.

## Operational Considerations
- Alerts on ingestion backlog, streaming error rates, and retention compliance.
- Provide runbooks for key rotation, Kafka rebalancing, and hotfix log deletion requests.
