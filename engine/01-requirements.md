# Engine Requirements

What the engine actually has to do, with rough sizing and operational
expectations.

## Workload catalog

### 1. Outbound message dispatch (HOT PATH)

The most frequent job. One job per outbound message.

- **Cardinality**: scales with traffic; at 1M msgs/mo = ~23 jobs/sec
  steady; bursts much higher
- **Latency target**: enqueue → dispatch in < 1 sec p99
- **Retries**: transient provider errors retried 3–5 times with backoff
- **Idempotency**: required — duplicate sends are an incident
- **Per-recipient or per-session ordering**: matters for WhatsApp
  (per-session sequential), doesn't for SMS

### 2. Webhook delivery to customers (HOT PATH)

One job per inbound or status event delivered to a customer webhook.

- **Cardinality**: ~2× outbound (delivery confirmation + reply)
- **Latency target**: provider → customer webhook in < 3 sec
- **Retries**: 3 tries with backoff: 0s, 30s, 5min, then dead-letter
- **HMAC signing**: required
- **Dead-letter**: customer-visible UI to inspect failed deliveries

### 3. WhatsApp session lifecycle (BACKGROUND)

Not really a "job" — it's a long-lived loop per session. The engine
isn't where the WebSocket lives, but it does run:

- **Session start/stop** triggered jobs (when a customer pairs/unpairs)
- **Auto-reconnect** loops (typically lives inside the worker process,
  not the engine, because the WebSocket has to)
- **Health-check ping** — every 60s per session

Distinction matters: engine dispatches start/stop; session worker holds
live state. Engine != session host.

### 4. Warmup schedule (DURABLE / CRON-LIKE)

For a new WhatsApp session, gradually raise the rate limit over 1–2
weeks.

- Day 1–3: 20 msg/day cap
- Day 4–7: 100 msg/day cap
- Day 8–14: 500 msg/day cap
- Day 15+: unlimited within rate

Implemented as a daily-tick job per session that bumps a config value.
Or as a **multi-step durable workflow**: `warmupWorkflow.start(sessionId)`
that sleeps and progresses over 14 real-world days.

This is where durable workflows shine. BullMQ alone can do it via
scheduled jobs but the code is uglier.

### 5. Periodic aggregations (CRON)

- Reputation scoring per number (every 1h)
- Quota/usage rollup per workspace (every 15m)
- Stale-session cleanup (every 6h)

Pure cron territory. Any cron-capable system handles this.

### 6. Compliance jobs

- Process inbound STOP keywords → add to opt-out list (event-driven)
- Refresh opt-out lists from external sources (cron daily)
- Audit log compaction (cron weekly)

### 7. Fanout (lower frequency, complex)

When a customer sends "broadcast this template to these 10,000 numbers,"
we need to:

- Validate consent for each recipient
- Apply per-session rate limits across all sends
- Stagger to avoid bursts
- Aggregate completion status

Durable-workflow shape. BullMQ-style queue with custom orchestration
would also work but it's more code.

## Throughput sizing (rough)

| Scenario | Outbound msgs/sec | Webhook deliveries/sec | Cron jobs |
|---|---|---|---|
| Hobby / Day 1 | < 1 | < 2 | A few/hour |
| Show HN spike | 10–100 transient | 20–200 transient | Few/hour |
| Year-1 SMB | ~10 sustained | ~20 sustained | A few/min |
| Year-3 mid-market | ~100 sustained | ~200 sustained | Many/min |

Mostly modest. Engine choice doesn't need to handle Snapchat scale; it
needs to handle "hundreds of msg/sec" with room to grow.

## DX requirements

What we want from the engine, ranked:

1. **TypeScript-native task definition.** We define a job in TS, get
   types end-to-end.
2. **Good local dev story.** Run locally without infra ceremony.
3. **Observable.** Built-in dashboard or at least clear logs per job
   execution.
4. **Cron and delayed jobs first-class.** Not a side-feature.
5. **Retries with backoff.** Configurable.
6. **Dead-letter handling.** Either built-in or easy to add.
7. **Concurrency controls.** Per-queue or per-key limits.
8. **Pausable / durable workflows.** Nice-to-have; mandatory for fanout
   and warmup if we want clean code.

## Operability requirements

- **Self-host friendly.** OSS users need to run this themselves.
- **Single-binary or simple compose-up.** No 5-service tangle.
- **Survives crashes.** Jobs don't get lost if a worker dies.
- **Postgres + Redis as backing store is fine.** We're already running
  both.

## Anti-requirements

What the engine does NOT need to be:

- **A WebSocket host.** That's the session worker; different concern.
- **A request-response framework.** That's our HTTP API.
- **A data store.** Postgres is.
- **A cache.** Redis is.

Keep the engine's scope tight.

## How this informs evaluation

The next docs evaluate Trigger.dev, then alternatives, against this
catalog. The decision matrix in `04-recommendation.md` scores each
option for how well it handles:

- Hot path dispatch (high throughput, low overhead)
- Durable workflows (warmup, fanout)
- Cron jobs
- DX
- Self-host
- Operational complexity
