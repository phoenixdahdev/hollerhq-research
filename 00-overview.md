# Research Overview — Multi-Channel Messaging Platform (SMS + WhatsApp)

> Status: Research phase. No implementation yet.
> Last updated: 2026-05-19
> Scope: SMS + WhatsApp as co-primary channels. Email is a possible
> Phase 4+ extension; not in scope yet.

## Why we're doing this

The project is a developer-grade SDK and runtime for sending and
receiving messages across **SMS and WhatsApp from one API**.

The discovery of [OpenWA](https://github.com/rmyndharis/OpenWA)
reframed our thinking toward WhatsApp as a first-class channel. But
SMS isn't an afterthought: it's the **legitimacy layer** — universally
delivered, regulated, the default for OTP / transactional / legal use
cases. WhatsApp is the **engagement layer** — richer media, read
receipts, much cheaper per conversation (often free), but doesn't
reach the ~10% of users without WhatsApp installed.

The two channels are complementary, not substitutes. A serious
messaging platform in 2026 needs both. The opportunity is making them
feel like one.

This research checks how big the opportunity is, how hard it is to
execute, and what the smallest meaningful product looks like.

## Core questions

1. **What's the SMS landscape in 2026?** Which provider(s) do we sit
   on top of?
2. **What's the WhatsApp landscape and which integration path?**
   Cloud API (official) vs unofficial (Baileys / OpenWA) vs both.
3. **What does a unified SMS + WhatsApp API actually look like?** The
   real product question. Smart routing, fallback, conversation
   continuity.
4. **Where is the market gap?** "Another messaging API" isn't a
   product.
5. **What's the architecture?** Sessions (WA) vs stateless calls
   (SMS), queueing, webhooks, multi-tenant.
6. **What's the engine for background jobs + workflows?** Trigger.dev
   and alternatives (see `engine/`).
7. **What's the storage and deployment story?** Postgres, Redis,
   object storage, hosting target (see `storage/`, `deployment/`).
8. **What are the risks?** WhatsApp ToS, account bans, SMS regulatory
   regimes (A2P 10DLC, TCPA, GDPR).
9. **How does "road to getting seen" play out?** Wedge into developer
   mindshare.

## Top-level document map

| # | File | What it covers |
|---|------|----------------|
| 00 | `00-overview.md` | This file — the map and the open questions |
| 01 | `01-sms-landscape.md` | SMS providers, A2P 10DLC, TCPA, pricing, the DX gap |
| 02 | `02-whatsapp-integration-options.md` | Cloud API vs BSPs vs unofficial — the WhatsApp paths |
| 03 | `03-openwa-deep-dive.md` | What OpenWA is, how it works, where it shines and breaks |
| 04 | `04-whatsapp-library-comparison.md` | Baileys vs OpenWA vs whatsapp-web.js |
| 05 | `05-multi-channel-strategy.md` | **The product idea**: unified SDK, routing, fallback, continuity |
| 06 | `06-architecture-and-sessions.md` | Multi-tenant architecture across both channels |
| 07 | `07-risks-and-mitigation.md` | WhatsApp ToS, ban risk, SMS regs (TCPA, A2P 10DLC), GDPR |
| 08 | `08-competitive-landscape.md` | SMS providers, WhatsApp SaaS, multi-channel platforms, notification infra |
| 09 | `09-recommended-path.md` | Synthesis — what we should actually build |

## Subfolders — focused implementation research

| Folder | Focus | Key recommendation |
|---|---|---|
| [`engine/`](./engine/00-overview.md) | Background jobs, durable workflows, scheduled tasks. Trigger.dev + alternatives (Inngest, BullMQ, Temporal, Hatchet) | Trigger.dev as primary engine; OSS, TS-native, self-hostable, durable workflows |
| [`storage/`](./storage/00-overview.md) | Postgres + ORM (Drizzle vs Prisma vs Kysely), Redis, object storage for media | Postgres + Drizzle; Redis; Cloudflare R2 (or MinIO for self-host) |
| [`deployment/`](./deployment/00-overview.md) | Runtime requirements, hosting (Fly.io, Railway, Render, K8s, VPS), self-host distribution | Fly.io for our SaaS; Docker compose on Hetzner VPS for OSS users |

## Suggested reading order

- **20-minute version**: `00 → 05 → 09 → engine/04-recommendation.md`.
  Frames the project, lands the product idea, surfaces the decisions,
  shows the engine choice.
- **Full version**: 00 → 09 in order, then subfolders. Files 01–04 are
  channel background, 05 is where the product idea lands, 06–08 are
  how-to and competitive context, 09 is decisions, subfolders are
  implementation-specific.

## What's intentionally out of scope (for now)

- Email channel (Phase 4+ if SMS + WhatsApp work)
- Voice / phone calls (different category — Twilio Voice, Vonage
  Voice)
- Push notifications (different category — see Novu / Knock)
- Pricing / business-model details (depends on path)
- Detailed API design — comes after we pick a direction
- Marketing plan beyond "what wedge gets devs to notice"
- Observability stack (Prometheus / Grafana / Sentry) — comes after
  implementation choice is locked
- Security details (KMS, key rotation, webhook HMAC) — covered partly
  in risks doc; full design after implementation
- Frontend stack for the dashboard — Next.js is assumed, details
  later

## How to use this folder

Read 00 → 09 in order, then dip into subfolders as their topics
become relevant. Files 01–08 + subfolder 01s are background and
analysis; 09 and subfolder `*-recommendation.md` are where decisions
land. After reading, the next step is a sit-down to answer the open
questions, then write a `decision.md` (per area or one combined) that
locks the direction before any code is written.

## Reading time

| Section | ~Reading time |
|---|---|
| Top-level 00–09 | ~80 min |
| `engine/` | ~25 min |
| `storage/` | ~15 min |
| `deployment/` | ~15 min |
| **Total** | **~135 min** |
