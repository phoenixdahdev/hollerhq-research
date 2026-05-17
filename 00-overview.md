# Research Overview — Multi-Channel Messaging Platform (SMS + WhatsApp)

> Status: Research phase. No implementation yet.
> Last updated: 2026-05-17
> Scope: SMS + WhatsApp as co-primary channels. Email is a possible
> Phase 4+ extension; not in scope yet.

## Why we're doing this

The project is a developer-grade SDK and runtime for sending and receiving
messages across **SMS and WhatsApp from one API**.

The discovery of [OpenWA](https://github.com/rmyndharis/OpenWA) reframed
our thinking toward WhatsApp as a first-class channel. But SMS isn't an
afterthought: it's the **legitimacy layer** — universally delivered,
regulated, the default for OTP / transactional / legal use cases.
WhatsApp is the **engagement layer** — richer media, read receipts, much
cheaper per conversation (often free), but doesn't reach the ~10% of
users without WhatsApp installed.

The two channels are complementary, not substitutes. A serious messaging
platform in 2026 needs both. The opportunity is making them feel like one.

This research checks how big the opportunity is, how hard it is to execute,
and what the smallest meaningful product looks like.

## Core questions

1. **What's the SMS landscape in 2026, and which provider(s) do we sit
   on top of?** Twilio, Telnyx, Plivo, Bandwidth, Vonage all sell SMS.
   Which one (or several) do we wrap, and how?
2. **What's the WhatsApp landscape and which integration path do we use?**
   Cloud API (official) vs unofficial (Baileys / OpenWA) vs both.
3. **What does a unified SMS + WhatsApp API actually look like?**
   This is the real product question. One `send()` call, two channels,
   plus smart routing / fallback / conversation continuity.
4. **Where is the market gap we can fill?** "Another messaging API"
   is not a product. What's specifically missing for developers building
   on top of SMS + WhatsApp today?
5. **What's the architecture?** Sessions (WhatsApp) vs stateless calls
   (SMS), queueing, webhooks, multi-tenant.
6. **What are the risks and how do we manage them?** WhatsApp ToS,
   account bans, SMS regulatory regimes (A2P 10DLC, TCPA, GDPR).
7. **How does "road to getting seen" play out?** What's our wedge into
   developer mindshare in a market with Twilio at one end and WhatsApp
   library forks at the other?

## Document map

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

## Suggested reading order

- **20-minute version**: `00 → 05 → 09`. Frames the project, lands the
  product idea, surfaces the decisions.
- **Full version**: 00 → 09 in order. Files 01–04 are channel background,
  05 is where the product idea lands, 06–08 are how-to and competitive
  context, 09 is decisions.

## What's intentionally out of scope (for now)

- Email channel (Phase 4+ if SMS + WhatsApp work)
- Voice / phone calls (different category — Twilio Voice, Vonage Voice)
- Push notifications (different category — different vendors, different
  DX, see Novu / Knock)
- Pricing / business-model details (depends on path)
- Detailed API design — comes after we pick a direction
- Specific hosting / cloud-provider decisions
- Marketing plan beyond "what wedge gets devs to notice"

## How to use this folder

Read 00 → 09 in order. Files 01–08 are background and analysis; 09 is
where decisions land. After reading, the next step is a sit-down to
answer the open questions in 09, then write a single `decision.md` that
locks the direction before any code is written.

## Reading time

| File | ~Reading time |
|---|---|
| 00 overview | 4 min |
| 01 SMS landscape | 8 min |
| 02 WhatsApp integration options | 10 min |
| 03 OpenWA deep dive | 8 min |
| 04 WhatsApp library comparison | 6 min |
| 05 Multi-channel strategy | 10 min |
| 06 Architecture & sessions | 12 min |
| 07 Risks & mitigation | 7 min |
| 08 Competitive landscape | 9 min |
| 09 Recommended path | 7 min |
| **Total** | **~80 min** |
