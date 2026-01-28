# 19. Designing & Implementing Photos Upload at Scale

## Problem Overview
- Accept billions of photo uploads globally with resumable transfers, malware scanning, and low-latency delivery via CDNs.

## Functional Requirements
- Support chunked uploads (multipart, tus protocol) and direct-to-cloud (signed URLs).
- Validate metadata, run antivirus + content moderation, and trigger async processing (thumbnails, ML tags).
- Store originals + derivatives across multi-region buckets with lifecycle policies.

## Non-Functional Goals
- Upload success > 99.95%, resumable chunk tolerance for flaky mobile networks.
- P95 end-to-end processing (upload -> thumbnails ready) < 2 minutes.
- Durability 11 nines with geo-redundancy; RPO ~0.

## Architecture Overview
- Upload gateway issues signed URLs, orchestrates chunk state machine, persists session metadata.
- Ingestion workers verify chunks, assemble objects in staging bucket, then copy to durable storage (S3/Blob store) with replication + erasure coding.
- Event bus triggers processing pipeline (Lambda/K8s) generating thumbnails, EXIF extraction, moderation tasks.
- CDN fronting derivative assets; metadata DB tracks versions + ACLs.

## Data Design & APIs
- Upload session table: `(upload_id, user_id, status, chunk_size, checksum, storage_path, expires_at)`.
- APIs: `POST /uploads`, `PUT /uploads/{id}/chunk`, `POST /uploads/{id}/complete`, `GET /media/{id}`.
- Metadata store captures derived artifact URIs + processing status.

## Implementation Plan
1. Build upload session service + signed URL issuance with IAM-scoped permissions.
2. Implement chunk assembler + checksum validation; persist to staging bucket + manifest.
3. Integrate malware/misuse scanners before promoting to canonical storage.
4. Launch async pipeline for thumbnails, metadata enrichment, CDN cache prewarming.
5. Configure lifecycle + GDPR deletion workflows, audit trails, and customer notifications.

## Testing & Validation
- Throttle network + drop chunks to validate resumable logic.
- Load test ingestion path to reach multi-GB/s throughput; monitor storage replication.
- Validate moderation hooks and failure retries.

## Operational Considerations
- Monitor upload success, chunk retries, processing backlog, CDN hit rate.
- Provide runbooks for partial failures (staging stuck, CDN purge requests, bucket failover).

## Tutorial Deep Dive
### Block Diagram
```mermaid
flowchart LR
    Clients --> Gateway[Upload Gateway\nSigned URLs/Auth]
    Gateway --> Staging[(Staging Bucket)]
    Gateway --> MetadataDB[(Metadata Service)]
    Staging --> Processing[Async Processing\n(Resizing, Moderation)]
    Processing --> Durable[(Durable Storage\nMulti-region)]
    Durable --> CDN[CDN/Edge]
    Processing --> Events[Event Bus for Notifications]
    Observability --> Gateway
    Observability --> Processing
```

### Design Walkthrough
- **Secure ingestion:** Clients request signed URLs, upload chunks directly to staging storage, and signal completion to metadata + orchestrator services.
- **Validation pipeline:** Antivirus, EXIF normalization, and policy checks run asynchronously before promoting objects to durable storage and CDN.
- **Derivative generation:** Resize/resample, add watermarks, and prewarm CDN caches; track per-derivative latency budgets.
- **Lifecycle:** Manage retention with storage class transitions, GDPR erasure flows, and audit logs for each mutation.

## Interview Kit
1. **How do you make uploads resilient on flaky mobile networks?**  
   Use resumable protocols (tus/multipart), include chunk checksums, and persist upload session state so retries pick up where they left off.
2. **What’s your plan for malware scanning at scale?**  
   Fan out files to scanning clusters, quarantine suspicious uploads, and integrate ML + signature engines; allow async release once clean.
3. **How do you handle CDN cache poisoning or purges?**  
   Require signed URLs, validate headers, and maintain purge automation that invalidates plus prewarms caches after updates.
