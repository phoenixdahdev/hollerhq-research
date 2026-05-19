# Engine Alternatives

Trigger.dev isn't the only option. Here's the field.

## Categorisation

- **Durable workflow platforms**: Trigger.dev (covered separately),
  Inngest, Temporal, Hatchet
- **Job queues** (lighter, lower-level): BullMQ, graphile-worker, River
- **Cloud-native serverless**: Vercel Workflow DevKit, Vercel Queues,
  Cloudflare Queues, AWS Step Functions
- **Legacy / out of scope**: Resque (Ruby), Sidekiq (Ruby), Celery
  (Python), Faktory

## Durable workflow platforms

### Inngest

[inngest.com](https://inngest.com). Closest competitor to Trigger.dev.

**Strengths:**

- TS-first, similar developer experience to Trigger
- Event-driven model (functions subscribe to events, not just direct
  triggers)
- Built-in concurrency, rate limiting, debounce, fairness — these are
  first-class primitives
- Excellent local dev: `npx inngest-cli dev` runs a UI
- Step functions for durable workflows
- Strong observability dashboard

**Weaknesses:**

- **Self-host story is limited.** The Inngest Dev Server is OSS for
  local development; the production server (Inngest Cloud) is largely
  proprietary. There IS an OSS server but it's noticeably behind cloud
  on features.
- For an OSS-first project wanting self-hostable engine, this is the
  killer issue.

**Fit:**

- If we were going SaaS-first, Inngest's hosted offering is excellent.
- For OSS-first, Trigger.dev wins on self-host parity.

### Temporal

[temporal.io](https://temporal.io). The heavyweight enterprise option.

**Strengths:**

- Battle-tested at huge scale (Snap, Datadog, Stripe internally)
- Multi-language (Go, TS, Java, Python, .NET, Ruby)
- Activities vs workflows distinction is clean
- Highest durability guarantees
- Open source (MIT)

**Weaknesses:**

- Operationally heavy. Cluster of services. Cassandra-or-Postgres
  database. Not "compose up and go."
- DX is more complex — multiple concepts (workflows, activities,
  signals, queries, retry policies)
- TS SDK is solid but the platform feels Go/Java-first

**Fit:**

- Overkill for V1
- If we ever hit Discord-scale volume, worth revisiting
- Not a good OSS-recommendation default because operating it is hard

### Hatchet

[hatchet.run](https://hatchet.run). Newer (2024). OSS (MIT).

**Strengths:**

- Postgres-only (no Redis required)
- Distributed task queue with DAG support
- Concurrency control, rate limiting
- Go core with TS SDK
- Easier to self-host than Temporal
- Focused, well-engineered

**Weaknesses:**

- Newer; smaller community
- TS SDK is good but the platform is Go-first
- Less polished dashboard than Trigger.dev / Inngest

**Fit:**

- Solid alternative if we want Postgres-only and don't mind a newer
  community
- Trigger.dev is more TS-native; Hatchet is more polyglot

## Job queues (lighter weight)

### BullMQ

[docs.bullmq.io](https://docs.bullmq.io). The default.

**Strengths:**

- Pure Node.js + Redis
- Battle-tested, mature, widely deployed
- Excellent docs
- Job priorities, delays, retries, scheduled jobs, rate limits
- Bull Board for a dashboard (separate package)
- Tiny operational footprint — just Redis + worker processes
- High throughput

**Weaknesses:**

- No native durable workflows (no `await wait.for` shape)
- Observability is bring-your-own
- Long-running multi-step workflows are awkward (compose jobs, manage
  state ourselves)
- Dead-letter handling is manual (failed jobs sit in `failed` set)

**Fit:**

- Excellent for the hot-path (outbound message dispatch)
- Painful for warmup schedules and fanout (you'd build a state
  machine yourself)
- Could be the entire engine if we accept the workflow ergonomics

### graphile-worker

[github.com/graphile/worker](https://github.com/graphile/worker).
Postgres-native job queue.

**Strengths:**

- Postgres only — one fewer system to run
- Crash-safe, transactional
- Cron support
- Used in production by serious shops
- Active maintenance

**Weaknesses:**

- Same workflow limitations as BullMQ (no durable workflows)
- Lower throughput ceiling than Redis-based queues at very high scale
- Observability is bring-your-own

**Fit:**

- Strong if we want to eliminate Redis (we won't — Trigger.dev needs
  Redis anyway, and we want it for session rate-limit counters)
- Comparable to BullMQ; both are good fits as "the queue"

### River

[riverqueue.com](https://riverqueue.com). Postgres-native, Go-first.

**Strengths:**

- Inspired by Sidekiq, very fast
- Postgres-only
- Strong Go ecosystem

**Weaknesses:**

- **TS SDK is community / incomplete.** Don't rely on it for a TS-first
  project.

**Fit:**

- Skip — TS support isn't there.

## Cloud-native serverless

### Vercel Workflow DevKit (WDK)

[vercel.com/docs/workflow-development-kit](https://vercel.com). Vercel's
new durable-workflow runtime (2025).

**Strengths:**

- Step-based, pause/resume, crash-safe
- Native integration with Vercel platform
- Good DX

**Weaknesses (for us):**

- **Tightly coupled to Vercel runtime.** Self-host parity isn't there.
- **Vercel can't host the long-lived WhatsApp WebSocket sessions** —
  serverless functions don't support persistent connections.
- We could use Vercel WDK for the engine while running WA sessions
  elsewhere, but that's two infras.

**Fit:**

- Skip — wrong runtime model for our use case.

### Vercel Queues

[vercel.com/docs/queues](https://vercel.com). Public beta (2025).

**Strengths:**

- Cheap ($0.60 / 1M ops)
- Durable, simple API
- Vercel-native

**Weaknesses (for us):**

- Same Vercel-coupling problem as WDK
- Less mature than alternatives

**Fit:**

- Skip — same reason.

### Cloudflare Queues + Workers

**Strengths:**

- Cheap, edge-distributed
- Workers ecosystem is fast

**Weaknesses (for us):**

- Workers have CPU time limits
- No persistent WebSocket connections (Durable Objects work but add
  complexity and don't fit our model)
- Self-host story = "use Cloudflare"

**Fit:**

- Skip.

### AWS Step Functions

**Strengths:**

- Durable
- Production-grade

**Weaknesses (for us):**

- AWS lock-in
- JSON DSL-driven (not TS-native)
- Cost adds up
- Not OSS

**Fit:**

- Skip for OSS project.

## What we exclude entirely

- Rails-era stuff (Sidekiq, Resque) — Ruby
- Python-era stuff (Celery) — Python
- Faktory — Ruby workers, less momentum in TS world
- Anything proprietary closed-source where self-host parity isn't
  possible (already excluded Inngest's prod stack for this reason)

## Quick comparison

| Tool | License | Stack | Durable WF | Self-host | TS DX | Op complexity |
|---|---|---|---|---|---|---|
| **Trigger.dev** | Apache 2.0 | Postgres + Redis | Yes | Yes (3 svcs) | Excellent | Medium |
| **Inngest** | Mixed | Postgres + Redis | Yes | Limited | Excellent | Low (cloud) / High (self) |
| **Temporal** | MIT | Cassandra/Postgres | Yes | Yes (cluster) | Good | High |
| **Hatchet** | MIT | Postgres | Yes | Yes | Good | Medium |
| **BullMQ** | MIT | Redis + Node | No | Yes (just Redis) | OK | Low |
| **graphile-worker** | MIT | Postgres + Node | No | Yes (just Postgres) | OK | Low |
| **Vercel WDK** | Proprietary | Vercel runtime | Yes | No | Excellent | Low (Vercel) |

## Honourable mentions

- **Knative + Kafka-based event streaming** — too heavy for our use case
- **PgBoss** (Postgres job queue) — less polish than graphile-worker
- **Quirrel** — cloud-only successor (defunct in OSS form)
- **Defer.run** — closed source, less self-host friendly

## Verdict

The real contest for us:

- **Trigger.dev** for durable + observable + self-host
- **BullMQ** for raw simplicity and throughput
- **Hatchet** as a darkhorse if we prefer Postgres-only

Decision lives in `04-recommendation.md`.
