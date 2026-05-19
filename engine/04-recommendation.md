# Engine Recommendation

## Headline

**Use Trigger.dev as the primary engine. Defer adding BullMQ unless
and until measured throughput problems show up.**

## Why this shape

- **Durable workflows matter more than raw throughput** for our use
  case. Warmup schedules, fanout broadcasts, webhook delivery-with-DLQ
  are all multi-step durable shapes. Trigger.dev handles them with
  one-line primitives; BullMQ requires custom state machines.
- **Self-host parity is non-negotiable** for our OSS positioning.
  Trigger.dev v3 has real self-host. Inngest doesn't.
- **TS-native DX is a wedge for our SDK** — using a TS-native engine
  internally keeps the whole stack feeling crafted.
- **Postgres + Redis is already our stack.** Trigger.dev sits on the
  same backing services. Zero extra storage to introduce.
- **One engine = one place to look** when something goes wrong.

## Why not the alternatives

- **BullMQ alone**: too thin for warmup / fanout; we'd reinvent
  durable-workflow primitives badly. Good as a supplement later if
  hot-path throughput demands it.
- **Inngest**: limited self-host kills it for OSS.
- **Temporal**: overkill operationally. Revisit at Discord scale.
- **Hatchet**: solid runner-up but smaller community, Go-first.
  Consider if Trigger.dev disappoints.
- **Serverless (Vercel WDK, CF, AWS)**: wrong runtime for our
  WebSocket-heavy workload.

## How it fits the architecture

```
┌──────────────────────────────────────────────────────────────────┐
│  API (Hono on Node, apps/api)                                      │
│      │                                                             │
│      ▼ trigger task                                                │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  Trigger.dev engine                                          │    │
│  │  - sendOutboundMessage(task)                                 │    │
│  │  - deliverCustomerWebhook(task)                              │    │
│  │  - warmupSession(durable workflow with wait.for)             │    │
│  │  - fanoutBroadcast(durable workflow)                         │    │
│  │  - cron: reputationScoreUpdate (every 1h)                    │    │
│  │  - cron: quotaRollup (every 15m)                             │    │
│  └────────────────────────────────────────────────────────────┘    │
│                              │                                     │
│                              ▼ internal RPC                        │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  Session worker (apps/worker)                                │    │
│  │  - holds WhatsApp sessions (persistent WebSocket)            │    │
│  │  - exposes internal RPC: enqueueWaSend(sessionId, msg)       │    │
│  │  - handles auto-reconnect, health pings                      │    │
│  └────────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────┘
```

Critical separation: **engine dispatches; session worker holds state.**
The WhatsApp WebSocket lives in the session worker, NOT in a Trigger.dev
task. Tasks are short-lived; sockets are not. The task
`sendWhatsAppMessage` calls into the session worker via an internal
RPC (HTTP or Redis pubsub) and waits for the result.

## Decision matrix scored against our needs

| Requirement | Weight | Trigger.dev | BullMQ | Inngest | Hatchet |
|---|---|---|---|---|---|
| Hot path throughput | M | 3/5 | 5/5 | 3/5 | 4/5 |
| Durable workflows | H | 5/5 | 1/5 | 5/5 | 5/5 |
| Cron / scheduled | H | 5/5 | 3/5 | 5/5 | 4/5 |
| Self-host parity | H | 5/5 | 5/5 | 2/5 | 5/5 |
| TS DX | H | 5/5 | 3/5 | 5/5 | 3/5 |
| Op complexity | M | 3/5 | 5/5 | 2/5 (self-host) | 3/5 |
| Observability built-in | M | 5/5 | 2/5 | 5/5 | 3/5 |
| Community size | L | 3/5 | 5/5 | 3/5 | 2/5 |

Weighted: Trigger.dev wins on the high-weight items by a clear margin.

## Phased plan

### Phase 0 (Foundation, ~half day)

- Add Trigger.dev to monorepo: `@your-org/jobs` package, holding task
  definitions
- Local dev: `npx trigger.dev@latest dev` runs alongside `pnpm dev`
- Define stub tasks for the workloads in `01-requirements.md` (just
  stubs, no implementations)

### Phase 1 (alongside main project Phase 1)

- Implement `sendOutboundMessage` task using Telnyx SMS driver
- API enqueues via `sendOutboundMessage.trigger({ messageId })`
- Implement `deliverCustomerWebhook` task with retry + DLQ

### Phase 2 (alongside main project Phase 2)

- Implement `sendWhatsAppMessage` task — RPCs into the session worker
- Implement `warmupSession` durable workflow
- Implement `healthCheckSession` cron

### Phase 3 (alongside main project Phase 3)

- `fanoutBroadcast` durable workflow
- Periodic aggregations (reputation, quota rollups)

### Phase 6+ (optimisation, if needed)

- If sustained > 100 msg/sec and per-task overhead becomes the
  bottleneck, introduce BullMQ for the hot path; keep Trigger.dev for
  durable workflows.

## Open questions

1. **Cloud Trigger.dev vs self-host from day one?**
   - For OSS users: self-host docs + Docker compose example
   - For our internal dev: use Trigger.dev Cloud free tier (5k runs/mo)
     to skip running it locally
   - Production for our SaaS: probably self-host for control
2. **Where does the session worker live?** Inside the same Node
   process that runs Trigger.dev task runners, or separate process?
   Lean: separate. Different scaling characteristics, different
   failure profiles.
3. **How do tasks call the session worker?** Internal HTTP API (simple,
   visible), Redis pubsub (faster), or in-process call (only works if
   colocated). Lean: internal HTTP API for clarity in Phase 2;
   optimize later if RPC overhead matters.
4. **Do we expose tasks to customers?** No. Trigger.dev is internal
   plumbing; customers see our HTTP API + webhooks only.

## What to read after this

When ready to commit on the engine, the next artifact is a one-page
`engine/decision.md` that locks the choice and the phase plan, ready
for implementation.

## TL;DR

> Trigger.dev as the only engine for V1. Apache-2.0, TS-native,
> self-hostable, durable workflows are best-in-class, Postgres + Redis
> backing matches our stack. Defer BullMQ. Skip Inngest (self-host
> story is bad), Temporal (overkill), serverless options (wrong
> runtime model for WebSockets).
