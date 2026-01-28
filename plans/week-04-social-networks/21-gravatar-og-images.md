# 21. Designing Gravatar & Dynamic OG Images

## Problem Overview
- Generate avatars or open-graph preview images on demand, supporting personalization and aggressive caching.

## Functional Requirements
- Hash-based lookup for default avatars + user-uploaded variants with deduplication.
- Dynamic rendering service (HTML/SVG to image) with template parameters.
- CDN caching with cache-busting controls when avatars change.
- Abuse/threat mitigation (rate limits, image sanitization).

## Non-Functional Goals
- P95 render latency < 100 ms for cached templates, < 300 ms for cold renders.
- Meet 99.99% availability with automated failover.

## Architecture Overview
- Request hits CDN; if miss, origin service queries metadata store, fetches assets, renders via headless renderer, stores derivative in cache bucket.
- Background invalidation triggers when user updates avatar; CDN purges + prewarms.
- OG image requests allow query parameters (title, subtitle, theme) validated against allowlist.

## Data Design & APIs
- Metadata: `(user_hash, asset_uri, etag, last_updated, privacy_flag)`.
- Rendering templates stored in versioned repo; config toggles customizing typography/colors.
- APIs: `GET /avatar/{hash}?size=`, `GET /og-image?title&theme`, `POST /avatar` for uploads.

## Implementation Plan
1. Implement hashing + upload pipeline with dedupe + moderation.
2. Build rendering service using headless Chrome/Puppeteer or Rust SVG renderer; add sandboxing.
3. Integrate CDN caching strategy (cache key includes template params) + soft TTL.
4. Create invalidation service for updates + scheduled refresh tasks.
5. Add metrics + tracing for render latency, cache hit, template errors.

## Testing & Validation
- Load test templating service with varied params; ensure CPU/memory safe.
- Security review for template injection, SSRF, and user data redaction.
- Validate CDN purges propagate globally within SLA.

## Operational Considerations
- Monitor render queue depth, template version drift, CDN error rates.
- Provide runbooks for rolling out new templates, emergency disabling of buggy templates, and abuse handling.

## Tutorial Deep Dive
### Block Diagram
```mermaid
flowchart LR
    Requests --> CDN[CDN Edge]
    CDN --> Renderer[Rendering Service\n(SVG/HTML -> Image)]
    Renderer --> AssetStore[(Asset Storage)]
    Renderer --> Templates[(Template Repo)]
    AssetStore --> CDN
    Renderer --> Cache[(Mid-tier Cache)]
    Security[Rate Limits + Sanitizers] --> Renderer
    Observability --> Renderer
```

### Design Walkthrough
- **Lookup & dedupe:** Resolve hashes to stored avatars, deduplicate uploads via hashing, and sanitize user uploads.
- **Rendering engine:** Generate OG images dynamically with templating, fonts, and localization; sandbox renderers to prevent SSRF or code injection.
- **Caching:** Store rendered outputs in mid-tier cache + CDN; include query params in cache key and allow purge triggers when users update avatars.
- **Resiliency:** Provide fallback avatars/templates if rendering fails and log incidents for quick rollbacks.

## Interview Kit
1. **How do you balance personalization with cache efficiency?**  
   Parameterize only meaningful inputs, bucket styles, and rely on signed URLs to avoid exposing secrets while still enabling caching.
2. **What protects rendering services from abuse?**  
   Enforce rate limits per user/IP, sanitize template inputs, run renders in jailed environments, and monitor CPU/memory anomalies.
3. **How do you roll out a new template safely?**  
   Canary with low-traffic cohort, compare render latency/error rates, and keep ability to revert template repo commit instantly.
