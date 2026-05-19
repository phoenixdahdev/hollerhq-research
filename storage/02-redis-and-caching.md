# Redis & Caching

## What we use Redis for

1. **Trigger.dev backing** — Trigger.dev uses Redis for its real-time
   event loop and queue management.
2. **Per-session rate limit counters** — fast increment + TTL.
3. **Per-recipient channel preference cache** — "is this number on
   WhatsApp?" lookups.
4. **Ephemeral session state** — QR codes pending scan, pairing flows.
5. **Webhook idempotency keys** — dedupe with short TTL.

## Why not Postgres for these

- Counters under contention are slow in Postgres without row-locking
  tricks
- TTLs are awkward (cleanup jobs vs Redis's native expire)
- Redis is what Trigger.dev expects

## Redis host options

| Option | Pros | Cons |
|---|---|---|
| **Self-hosted (Docker)** | Free, simple | We manage |
| **Upstash** | Serverless, pay-per-request, REST API | Per-request cost adds up at scale; some Trigger.dev features need full Redis protocol |
| **Redis Cloud / Redis Enterprise** | Managed, full features | Costly, enterprise-flavoured |
| **AWS ElastiCache** | Managed | AWS coupling |
| **DragonflyDB** (Redis-compatible) | Faster, more efficient | Newer, less battle-tested |
| **KeyDB** (Redis fork) | Multithreaded | Less momentum, project pace uncertain |

**Lean for OSS users:** Docker compose Redis self-hosted.
**Lean for our SaaS:** managed Redis (Redis Cloud, DO Managed, or
Upstash if we confirm Trigger.dev compatibility).

## Caching patterns

### Channel-preference cache

```
Key:   channel-pref:{workspaceId}:{recipientPhone}
Value: { has_whatsapp: true, last_checked: ... }
TTL:   30 days
```

Set on first send to a recipient; updated on inbound; lazily refreshed.

### Rate limit counters

```
Key:   rate:{sessionId}:{minute_bucket}
Value: integer counter
TTL:   120 seconds
```

Pattern: INCR + EXPIRE on first set. Check value before send; refuse
if over limit. Conservative defaults documented in
`research/07-risks-and-mitigation.md`.

### Pairing flow ephemeral state

```
Key:   pairing:{sessionId}
Value: JSON with state (pending QR, scanned, expired)
TTL:   5 minutes
```

Replaced on each QR rotation.

### Webhook idempotency

```
Key:   webhook-idem:{webhookId}:{externalId}
Value: 1
TTL:   24 hours
```

Set on successful delivery; check before re-delivering on retry.

## Anti-patterns

Don't use Redis as primary storage for anything we'd cry about losing.
Redis can be flushed, evicted, or lost. Postgres is durable; Redis is
hot.

## Memory sizing

Rough budget for ~100 active WA sessions, 100k recipients cached:

| Use | Memory |
|---|---|
| Trigger.dev internal | ~100 MB |
| Channel-pref cache | ~10 MB |
| Rate counters | ~5 MB |
| Pairing state | < 1 MB |
| Webhook idem | ~10 MB |
| **Total** | **~150 MB** |

Fits comfortably on the smallest paid Upstash plan or a self-hosted
Redis on the same VPS as Postgres.
