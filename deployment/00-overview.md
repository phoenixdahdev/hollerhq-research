# Deployment Research — Overview

Where the messaging platform runs in production, and how OSS users
self-host it.

## What we need from a runtime

Our application has constraints not every platform supports:

1. **Long-lived processes** — WhatsApp sessions hold open WebSockets
   for hours / days. Serverless functions (5–30 min limits) don't fit.
2. **WebSocket support** — outbound to WhatsApp servers.
3. **Persistent disk OR equivalent** — session auth state (encrypted)
   stored in Postgres works, but some media caching benefits from local
   disk.
4. **Postgres + Redis** — managed or self-hosted nearby.
5. **Internal networking** — API ↔ session worker ↔ engine.
6. **Multiple processes** — API, worker, engine task runners,
   dashboard.
7. **Build pipeline** — Turborepo + Docker.
8. **Scaling** — horizontal worker scaling for SMS load, sticky
   sessions for WA.

## What this rules out (or makes hard)

- **Vercel** — serverless functions only; no persistent WS; can't host
  WA sessions.
- **Cloudflare Workers** — same problem; Durable Objects don't fit
  our model cleanly.
- **AWS Lambda** — same.
- **Netlify Functions** — same.

We need real long-running processes. That points to:

- **Fly.io** — runs Docker containers globally, persistent volumes, WS
  support, reasonable price
- **Railway** — friendly DX, runs containers
- **Render** — runs containers, managed PG
- **DigitalOcean App Platform** — runs containers
- **Self-hosted VPS** — Hetzner, OVH, etc.
- **Kubernetes** — for serious scale

## Document map

| # | File | What it covers |
|---|---|---|
| 00 | `00-overview.md` | This file |
| 01 | `01-runtime-requirements.md` | Detailed runtime needs and constraints |
| 02 | `02-hosting-options.md` | Fly.io, Railway, Render, K8s, VPS compared |
| 03 | `03-self-host-distribution.md` | Docker compose for OSS users — what we ship |

## Open questions

1. Hosting target for our own SaaS production (when we get there)?
2. Hosting target for our demo / Show HN env?
3. What's the recommended deployment for OSS users — single-VPS Docker
   compose, or production-grade with multiple machines?
