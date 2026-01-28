# 14. Designing Load Balancers Without a SPOF

## Problem Overview
- Provide horizontally scalable L4/L7 load balancing that survives component failures without traffic black holes.

## Functional Requirements
- Multi-layer architecture: Anycast/DNS -> edge L4 -> L7/application gateways.
- Health-checking + failover per hop, including config distribution.
- Support multiple routing algorithms (round robin, least connections, consistent hashing) with per-service configs.

## Non-Functional Goals
- Achieve 99.99% availability for gateway stack, zero single point of failure.
- Config propagation < 1 minute globally.
- Handle 10 Tbps aggregate throughput with DDoS protections.

## Architecture Overview
- Control plane replicates configs via consensus store (etcd/Raft) across regions.
- Data plane comprised of stateless Envoy/HAProxy pods fronted by Anycast IPs and BGP announcements.
- Observability via eBPF sampling, per-hop latency tracking, and synthetic probes.

## Data Design & APIs
- Config schema: `(service, protocol, ports, routing_policy, healthcheck, retries, timeouts)`.
- Admin APIs: `POST /config`, `POST /traffic-shift`, `GET /health`. Config changes go through staged rollouts.

## Implementation Plan
1. Establish multi-region Anycast network + DNS fallback.
2. Deploy L4 (e.g., Maglev) + L7 (Envoy) tiers with independent autoscaling + health.
3. Build control plane that validates configs, stores in etcd, and pushes via xDS.
4. Implement chaos automation (drain nodes, withdraw BGP) to verify no SPOF.
5. Document incident response + rollback flows for config pushes.

## Testing & Validation
- Run synthetic latency/availability checks from multiple geos.
- Simulate config errors; confirm canarying + automatic rollback.
- Validate failover when entire region offline (traffic shifts via Anycast/DNS).

## Operational Considerations
- Monitor BGP session health, control-plane replication lag, and per-service error rates.
- Maintain runbooks for cert rotation, config schema migrations, and DDoS mitigation steps.
