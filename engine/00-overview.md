# Engine Research — Overview

> Status: Research phase. No implementation yet.
> Scope: durable background-job / orchestration runtime for the
> messaging platform. Sometimes called the "engine," "workflow runtime,"
> or "job runner."

## What we mean by "engine"

The platform has work that doesn't fit in a single HTTP request:

- **Sending an outbound message** — queue, rate-limit, retry, observe
- **Delivering a webhook to a customer** — retry with backoff,
  dead-letter
- **Running a warmup schedule for a new WhatsApp session** — drip-feed
  for days
- **Periodic health checks** for each WA session — cron-like
- **Refreshing reputation scores** — periodic aggregation
- **Fanout sends** (broadcast within rate limits)

Anything that needs to be reliable, retried, observable, scheduled, or
durable across crashes. That's "the engine."

This is distinct from:

- **The HTTP API** — accepts requests, enqueues work, returns immediately
- **The WhatsApp session worker** — holds long-lived WebSocket per
  session
- **The database** — stores state
- **The cache** — short-lived hot data

The engine sits behind the API, dispatches to the session worker (or
direct to providers for SMS), and runs all the periodic / retried jobs.

## Why it matters

The single biggest determinant of operability for this kind of platform
is the job-running layer. Pick a heavy framework and we have ops burden
forever. Pick something too thin and we'll spend months rebuilding
observability, dashboards, dead-letter UIs.

Trigger.dev was the user's prompt. We should evaluate it seriously plus
the obvious open-source alternatives.

## Document map

| # | File | What it covers |
|---|---|---|
| 00 | `00-overview.md` | This file |
| 01 | `01-requirements.md` | Concrete background workloads; throughput; DX needs |
| 02 | `02-trigger-dev.md` | Trigger.dev v3 deep dive — self-host, model, fit |
| 03 | `03-alternatives.md` | Inngest, BullMQ, Temporal, Hatchet, graphile-worker, serverless options |
| 04 | `04-recommendation.md` | Decision matrix + recommended path |

## Key constraints (so the evaluations stay grounded)

- **OSS positioning** → self-host parity is non-negotiable
- **TS-first stack** → engine must have first-class TS SDK
- **Postgres + Redis already required** → engine can reuse, shouldn't
  add a third datastore
- **Persistent WebSocket workloads exist** but live in the **session
  worker**, NOT the engine — engine handles short-lived jobs; sockets
  live in a long-running worker process

## Open questions

1. One engine for everything, or split (e.g., BullMQ for hot, Trigger.dev
   for orchestration)?
2. How important is self-host parity with cloud? (For OSS adoption it's
   critical; for our own SaaS, less so.)
3. Postgres-native engine, Redis-native engine, or both backings?
4. How does the engine interact with the WA session worker? (Engine
   runs jobs; session worker holds live sockets — different concerns.)
