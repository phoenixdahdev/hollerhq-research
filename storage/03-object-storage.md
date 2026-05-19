# Object Storage

For media files received via WhatsApp (images, video, audio, documents)
and possibly large customer webhook payloads.

## Why we need it

- WhatsApp media can be tens of MB per file
- Storing in Postgres is expensive and slow
- Customers want to download media via a URL we generate

## Options

| Provider | Pros | Cons |
|---|---|---|
| **AWS S3** | Industry default, mature, widely supported | Egress costs ($$$ if customers download a lot) |
| **Cloudflare R2** | S3-compatible API, **zero egress fees**, cheap | Newer, fewer integrations |
| **Tigris** | S3-compatible, globally distributed, cheap | Newer still |
| **Backblaze B2** | Very cheap, S3-compatible | Less feature-rich |
| **MinIO** (self-hosted) | OSS, S3-compatible | We run it |
| **Supabase Storage** | OSS, built on S3 | Coupling to Supabase |

**Lean for OSS users:** docs default to S3-compatible config; example
compose includes MinIO for self-host.
**Lean for our SaaS:** Cloudflare R2 — zero egress is huge for media-
heavy traffic.

## Pattern

1. Driver receives media event
2. Driver downloads media from WhatsApp's CDN (often short-lived URL)
3. Driver uploads to our object storage
4. Driver records object key in `messages.body.media.object_key`
5. Customer accesses via our presigned URL (short-lived, signed)

## Security

- All objects private by default
- Access via presigned URLs only
- Per-workspace key prefixes (no cross-workspace access)
- Encryption at rest (server-side via storage provider)

## Lifecycle

- Inbound media retained per workspace retention policy (default 90
  days)
- Lifecycle rules in storage provider auto-delete
- Re-fetch from WhatsApp if needed within media's CDN window (~14
  days)

## Key naming scheme

```
{workspace_id}/{year}/{month}/{message_id}/{filename}

Example:
ws_abc/2026/05/msg_xyz/photo.jpg
```

Keeps per-workspace browsing easy and supports lifecycle rules at the
workspace level.

## Cost rough estimate

For ~100k inbound media items/mo, average 500 KB each = ~50 GB/mo
stored:

| Provider | Storage cost/mo | Egress (50 GB/mo) | Total |
|---|---|---|---|
| AWS S3 | ~$1.15 | ~$4.50 | ~$5.65 |
| Cloudflare R2 | ~$0.75 | $0 | ~$0.75 |
| Backblaze B2 | ~$0.30 | ~$0.50 | ~$0.80 |

R2 wins clearly once egress is a real number. For OSS users with MinIO,
both costs are essentially zero (storage = local disk).
