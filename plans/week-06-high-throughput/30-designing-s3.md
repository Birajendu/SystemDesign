# 30. Designing S3-like Object Storage

## Problem Overview
- Provide durable, elastic object storage with simple APIs, geo-redundancy, and lifecycle controls.

## Functional Requirements
- Namespace with buckets + objects supporting PUT/GET/HEAD/DELETE, multipart uploads, versioning, lifecycle rules.
- Separate control plane (auth, metadata) from data plane (storage nodes) for scalability.
- Support replication, cross-region bucket replication, and event notifications.

## Non-Functional Goals
- Durability 11 nines, availability 99.99%, strong read-after-write for new objects, eventual for overwrite.
- Scale to trillions of objects, exabytes of data with linear cost efficiency.

## Architecture Overview
- Control plane stores bucket/object metadata in Paxos/Raft-backed DB; handles IAM policies + encryption keys.
- Data plane writes objects into chunk servers using erasure coding + replication; background scrubbing verifies integrity.
- Front-end API gateways enforce auth, throttling, and route to nearest data plane region.

## Data Design & APIs
- Metadata: `(bucket, object_key, version_id, etag, size, storage_class, encryption, retention)`.
- Multipart upload manifests track parts; lifecycle policies stored per bucket.
- APIs for event notifications, replication configuration, and IAM policies.

## Implementation Plan
1. Design metadata sharding + consensus layout; implement bucket/object CRUD with versioning.
2. Build storage nodes with chunk maps, erasure coding, background scrubbing, and repair.
3. Implement front-end API + SDK compatibility (signature v4, presigned URLs).
4. Add lifecycle + replication engine, storage classes (standard, infrequent, glacier) with tiering workflows.
5. Instrument metrics (Durability checks, p95 latency, error budgets) and automation for capacity planning.

## Testing & Validation
- Fault-inject disk/node/region loss to verify durability + replication.
- Benchmark throughput for multipart upload, list operations, and small-object workloads.
- Validate IAM + encryption flows, ensure compliance logging.

## Operational Considerations
- Monitor metadata DB health, repair backlog, erasure coding CPU, S3 API throttling.
- Provide runbooks for bucket migrations, version purge operations, and incident response.
