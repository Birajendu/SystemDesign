# 13. Scaling WebSockets

## Problem Overview
- Maintain millions of persistent, bidirectional connections with predictable resource usage and graceful degradation.

## Functional Requirements
- Gateway pools that negotiate WebSocket upgrades, authenticate sessions, and proxy to backend services.
- Load balancers supporting sticky routing and connection draining for deployments.
- Backpressure handling, rate limiting, and fallback (long polling/SSE).

## Non-Functional Goals
- Target 5M concurrent connections per region, handshake success > 99.95%.
- Keep keep-alive latency < 30 ms and detect dead sockets within 30 seconds.

## Architecture Overview
- Anycast IP + L4 load balancers -> L7 proxies (Envoy/NGINX) -> WebSocket gateway pods.
- Control plane tracks connection placement, session metadata, and quotas.
- Observability pipeline collects connection counts, ping/pong latency, and error codes.

## Data Design & APIs
- Session registry storing `(connection_id, user_id, shard, backend_routing_key, last_ping)` in Redis/etcd.
- Admin APIs: `POST /drain`, `GET /metrics`, `POST /disconnect` for moderation/eject.

## Implementation Plan
1. Establish capacity model (CPU, memory per connection) and autoscaling rules.
2. Implement gateway service with TLS termination, compression negotiation, and heartbeat management.
3. Build routing metadata store + consistent hashing to minimize reconnect storms.
4. Add failover + global load balancing (multi-region) with warm standby.
5. Document client SDKs with exponential backoff, jitter, and fallback strategies.

## Testing & Validation
- Load test handshake surges + sustained connections using tools (h2load, vegeta).
- Kill batches of gateway pods; verify graceful failover and limited reconnect storms.
- Inject packet loss to tune heartbeat thresholds vs. false positives.

## Operational Considerations
- Monitor OS limits (file descriptors), TLS cert expiration, and autoscaler lag.
- Keep blue/green deployment process with connection draining + draining time budgets.
