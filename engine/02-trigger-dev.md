# Trigger.dev — Deep Dive

The user's named option. Worth a careful look.

## What it is

[Trigger.dev](https://trigger.dev) is an OSS background-jobs and durable-
workflow framework for TypeScript. Founded 2022, raised seed + Series A,
well-funded. Apache 2.0 licensed.

**v3** (late 2024) was a major rewrite. The platform is now:

- **TypeScript-first** with typed tasks
- **Durable** — tasks survive crashes via a write-ahead-log model
- **Self-hostable** — you can run the entire platform yourself
- **Cloud option** at [cloud.trigger.dev](https://cloud.trigger.dev) for
  managed
- Built on **Postgres + Redis** (familiar stack — same as ours)

## The model

Tasks are async TS functions. Each task call gets:

- A unique run ID
- Crash-safe execution (resumes from last step on restart)
- Built-in retries
- Built-in idempotency keys
- Observability in a built-in dashboard

```ts
import { task } from "@trigger.dev/sdk/v3";

export const sendOutboundMessage = task({
  id: "send-outbound-message",
  retry: { maxAttempts: 5, factor: 2, minTimeoutInMs: 1000 },
  run: async (payload: { messageId: string }) => {
    // ... do the work
    // automatically retried on throw
    // run state persisted at each `await`
  },
});

// Trigger from anywhere
await sendOutboundMessage.trigger({ messageId: "msg_abc" });
```

Durable steps via `wait.for`:

```ts
export const warmupSession = task({
  id: "warmup-session",
  run: async ({ sessionId }: { sessionId: string }) => {
    await db.session.setLimit(sessionId, 20);
    await wait.for({ days: 3 });         // pauses, frees resources
    await db.session.setLimit(sessionId, 100);
    await wait.for({ days: 4 });
    await db.session.setLimit(sessionId, 500);
    await wait.for({ days: 7 });
    await db.session.setLimit(sessionId, null);
  },
});
```

The `wait.for` literally pauses execution — process can restart, server
can crash, the task resumes from the right place when the time comes.
This is the durable-workflow superpower; very hard to achieve with raw
BullMQ.

## Strengths

- **TS-first DX.** Tasks are typed; payloads are typed; the dashboard
  shows real type information.
- **Durable workflows.** `wait.for`, `wait.until`, `idempotencyKey` are
  best-in-class.
- **Self-host parity.** Same code, same features, OSS vs cloud.
- **Observability.** Built-in dashboard with runs, logs, retries,
  payload inspection.
- **Concurrency.** Per-queue concurrency limits ("at most 5 of this
  task at once"), per-key concurrency (e.g., "at most 1 per session
  id").
- **Scheduled tasks.** Cron is first-class.
- **Funding + momentum.** Active development; not at risk of
  abandonment.

## Weaknesses / things to know

- **Operational footprint.** Self-host runs ~3 services (web,
  supervisor, run engine). Compose file is non-trivial. Not "drop in
  next to your existing app."
- **Overhead per task.** Each invocation has setup cost. For
  thousand-msg/sec hot-path use, raw BullMQ is faster per-job.
- **Lock-in flavour.** Tasks defined with Trigger's primitives;
  migrating away is non-trivial.
- **Newer than BullMQ.** Smaller community; fewer Stack Overflow
  answers.
- **v3 is a rewrite of v2.** Some 2023-era blog posts and SO answers
  refer to v2 patterns that no longer apply — be selective about
  search results.
- **Postgres + Redis required.** Same as our stack, so this is fine,
  but worth noting.

## Fit against our requirements

| Requirement | Fit |
|---|---|
| Hot path dispatch (outbound msg) | OK at low/mid scale; overhead-per-task at very high throughput |
| Durable workflows (warmup, fanout) | Excellent |
| Cron jobs | Excellent |
| Webhook delivery with retries + DLQ | Excellent |
| TS DX | Excellent |
| Local dev | Good (`npx trigger.dev@latest dev`) |
| Self-host friendly | OK (3 services to run) |
| Survives crashes | Yes — core promise |

## Pricing (cloud)

- Free: 5k runs/month, 10 concurrent
- Pro: $20/mo, 50k runs included, then $0.30/1k
- Scaling tiers above that

For us as developers building on top, cloud cost would be small. For
our customers who self-host? Free.

## Self-host story

- Docker-compose-able
- Requires Postgres + Redis (already in our stack)
- Three services: webapp (dashboard), supervisor (worker spawner),
  task runners (per-task processes)
- Detailed docs at
  [trigger.dev/docs/self-hosting](https://trigger.dev/docs/self-hosting)

Not trivial but well-documented.

## How we'd use it

Recommended split:

- **Trigger.dev for durable / scheduled / observable work**: warmup
  schedules, fanout broadcasts, periodic aggregations, webhook
  delivery with DLQ
- **BullMQ for raw hot-path dispatch**: outbound message dispatch jobs
  that are simple "submit and forget" — lower overhead, faster per
  job

Or **just Trigger.dev for everything** — simpler operationally, accept
the per-job overhead, and re-evaluate if throughput becomes a problem.

Lean toward the latter for V1 simplicity; split later if measured
cause arises. See `04-recommendation.md`.

## How to peek

```bash
# In a scratch dir
npx trigger.dev@latest init
# Defines a sample task
# Run the dev server
npx trigger.dev@latest dev
# Trigger from a Node script
```

Spend 30 minutes building a `sendSms` task that calls a mock SMS
sender, add retry, add a 5-second `wait.for`, watch how the dashboard
tracks runs. That hour is enough to judge fit.

## Verdict

**Strong candidate for our primary engine.** Best-in-class DX for the
durable-workflow shape, self-hostable, well-funded, TS-native. Main
risks are operational footprint and per-task overhead. Both are
manageable.

Detailed comparison with alternatives in `03-alternatives.md`.
