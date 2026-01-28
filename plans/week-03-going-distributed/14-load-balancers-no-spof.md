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

## Tutorial Deep Dive
### Block Diagram
```mermaid
flowchart LR
    Clients --> Anycast[Anycast/DNS]
    Anycast --> EdgeL4[L4 Edge Tier]
    EdgeL4 --> L7[L7 Gateways/Envoy]
    L7 --> Services[Application Services]
    Control[Control Plane\n(etcd/Raft + Config API)] --> EdgeL4
    Control --> L7
    Telemetry[Telemetry + Synthetic Probes] --> EdgeL4
    Telemetry --> L7
    EdgeL4 --> DDoS[DDoS Mitigation]
```

### Design Walkthrough
- **Control/data separation:** Configs, certificates, and health rules live in a replicated control plane that pushes versions via xDS-style APIs; data plane stays stateless.
- **Multi-layer routing:** Combine Anycast addresses, DNS failover, L4 VIPs, and L7 gateways so losing any component doesn’t block traffic entirely.
- **Health + canaries:** Synthetic checks and per-hop latency metrics trigger auto-draining; config pushes roll out in waves with fast rollback.
- **DDoS posture:** Keep dedicated scrubbing/edge tier, enforce connection limits, and integrate with upstream providers for volumetric attacks.

## Interview Kit
1. **How do you roll config without taking down the fleet?**  
   Validate in staging, push to 1% canary, monitor error budget, then roll globally; keep previous version cached for instant rollback.
2. **What happens when a region dies?**  
   Withdraw BGP announcements, fail DNS health checks, reroute clients to secondary regions, and scale there automatically while SREs investigate.
3. **How do you store per-tenant routing policies safely?**  
   Version configs, enforce schema validation, and restrict access via RBAC + approvals; include diff previews for human review.
