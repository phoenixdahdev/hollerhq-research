# Storage Research — Overview

What the messaging platform needs to store, and which datastores /
services to use.

## What we store

| Data | Volume | Access pattern | Suggested store |
|---|---|---|---|
| Workspaces, API keys | Small | OLTP | Postgres |
| WhatsApp sessions (auth state) | Medium per session, hundreds–thousands of sessions | Read on connect, write on creds update | Postgres (encrypted) |
| SMS provider configs | Small | OLTP | Postgres (encrypted) |
| Outbound messages | High volume, retained | OLTP + analytical | Postgres (partitioned later) |
| Inbound messages | High volume | OLTP | Postgres |
| Conversations | Medium | OLTP | Postgres |
| Webhooks | Small | OLTP | Postgres |
| Opt-out lists | Medium | Lookup on every send | Postgres + Redis cache |
| Engine state (Trigger.dev internals) | High | Managed by Trigger.dev | Postgres + Redis (Trigger.dev's tables) |
| Hot cache (per-recipient channel preference) | Medium | Read on every send | Redis |
| Per-session rate limit counters | High | Read/write per send | Redis |
| QR codes, ephemeral session state | Small | Short-lived | Redis with TTL |
| WhatsApp media (images, video, docs) | Variable, large | Read on send, store on receive | Object storage |
| Customer webhook payloads (audit) | Medium | Append-only | Postgres or object storage |

## Document map

| # | File | What it covers |
|---|---|---|
| 00 | `00-overview.md` | This file |
| 01 | `01-relational-database.md` | Postgres choice + ORM (Drizzle / Prisma / Kysely) |
| 02 | `02-redis-and-caching.md` | Redis for cache + rate limit + ephemeral state |
| 03 | `03-object-storage.md` | Media files: S3 / R2 / Tigris |

## Open questions

1. Which Postgres ORM? Drizzle vs Prisma vs Kysely
2. Postgres host? Self-host vs Neon vs Supabase vs Vercel Postgres
3. Redis host? Self-host vs Upstash vs Redis Cloud
4. Object storage? S3 vs Cloudflare R2 vs Tigris
5. How aggressively do we partition the `messages` table?
