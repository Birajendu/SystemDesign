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

## Tutorial Deep Dive
### Block Diagram
```mermaid
flowchart LR
    Clients --> Anycast[Anycast/DNS]
    Anycast --> L4[L4 Load Balancers]
    L4 --> Gateways[WebSocket Gateways]
    Gateways --> Services[Backend Services]
    Gateways --> Registry[Session Registry (Redis/etcd)]
    Services --> Stores[(Stateful Stores)]
    Observability --> Gateways
    Observability --> Services
```

### Design Walkthrough
- **Entry tier:** Use Anycast/DNS plus L4 balancers to steer clients into regional pools; enforce TLS termination and upgrade limits here.
- **Gateway behavior:** Maintain connection state, heartbeats, and message routing; rely on session registry for global disconnects or targeted messaging.
- **Scaling:** Model CPU/memory per connection, pre-scale pools ahead of traffic spikes, and drain updates during deploys using connection migration.
- **Fallbacks:** Provide long-polling/SSE fallback for clients that can’t reconnect, and degrade gracefully by limiting optional channels first.

## Interview Kit
1. **How do you prevent reconnect storms after an outage?**  
   Use randomized backoff in clients, stagger gateway restarts, and keep warm standby capacity to absorb surges.
2. **What metrics prove gateway health?**  
   Connection count, handshake success rate, ping/pong latency, CPU/memory per node, and error distribution across opcodes.
3. **How would you evict a single user across the fleet?**  
   Look up connection metadata in the registry, push a disconnect command via control channel, and confirm ack/cleanup to avoid ghost sessions.
