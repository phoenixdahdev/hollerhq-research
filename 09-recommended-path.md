# Recommended Path

Synthesis of everything in this folder, plus the questions we still need
to answer before writing code.

## Headline recommendation

**Build an open-source, TypeScript-first multi-channel messaging SDK
with SMS and WhatsApp as co-primary channels, unified under one API with
smart channel routing. Wrap one SMS provider and one WhatsApp driver to
start, abstract over both behind a `MessageDriver` interface. Ship the
SDK first, the hosted runtime second.**

## Why this shape

- **Multi-channel from Day 1.** Shipping WhatsApp-only or SMS-only is
  the wrong wedge. The **unified API is the product**. See
  `05-multi-channel-strategy.md`.
- **TypeScript-first.** The messaging ecosystem is full of partial TS,
  auto-generated types, and "good enough" definitions. Hand-crafted
  strict types is a real DX wedge.
- **One SMS provider + one WhatsApp driver to start.** Telnyx for SMS
  (cost + DX), Baileys for WhatsApp (lightweight + permissive license).
  Both via the unified `MessageDriver` interface so additional providers
  are pluggable.
- **SDK before hosted.** The SDK is what gets GitHub stars and developer
  attention; the hosted runtime is the business model. Community first.

## What this does NOT mean

- We don't try to be Twilio. Their value-add is global infrastructure +
  sales motion. Ours is DX + multi-channel design.
- We don't try to be Wassenger / Chat-API. Their value-add is hosting;
  ours is the library + design quality.
- We don't compete with Novu / Knock for notification infrastructure.
  Those are one-way; we're conversational.
- We don't build a chatbot framework (Typebot, Botpress). We're
  infrastructure under chatbot frameworks.

## Phased plan (research-stage only, not commitments)

### Phase 0: Foundation (1 weekend)

- Turborepo monorepo
- Stub packages: `@your-org/types`, `@your-org/sdk-node`, `apps/api`,
  `apps/dashboard`
- Internal packages: `@your-org/messaging-core` (interfaces, router,
  conversation model), `@your-org/sms-driver-telnyx`,
  `@your-org/whatsapp-driver-baileys`, `@your-org/messaging-mocks`

### Phase 1: SMS vertical slice (1 weekend — safer first step)

- Telnyx driver implementing `MessageDriver`
- API endpoint: `POST /messages` (channel = `'sms'`)
- Webhook handler for delivery receipts and inbound SMS
- Conversation threading by recipient phone
- One end-to-end flow: send SMS, receive reply, fire webhook
- **Why SMS first**: simpler (no sessions, no QR), faster to working
  end-to-end, validates the unified API design before adding WhatsApp
  complexity

### Phase 2: WhatsApp via Baileys (1–2 weekends)

- Baileys driver implementing `MessageDriver` (with optional session
  methods)
- Pairing flow (QR + phone-pairing code)
- WhatsApp message in / out, threaded into same conversation model as
  SMS
- Unified webhook event shape across both channels
- End-to-end: pair WA number, send via either channel, see one
  conversation

### Phase 3: Router + fallback (1 weekend)

- `messaging.send()` with explicit channel
- `messaging.send()` with channel array (try-in-order fallback)
- Internal channel router with hooks for smart routing

### Phase 4: SDK polish + Show HN (1 weekend)

- Clean TS types on `@your-org/sdk-node`
- Real README, quickstart, code examples for both SMS and WA
- Publish to npm as v0.1
- 60-second demo video
- Show HN with "open-source messaging SDK: SMS + WhatsApp, unified"

### Phase 5: Cloud API driver (2 weekends)

- WhatsApp Cloud API driver as alternative to Baileys
- Document trade-offs between drivers in docs
- Same `MessageDriver` interface

### Phase 6: Smart routing (2 weekends)

- Track conversation history per recipient × channel
- Implement smart routing strategy (prefer WA if in 24h window, etc.)
- Add cost estimation API

### Phase 7: Hosted runtime (open question)

- Decide based on Phase 4 traction
- If yes: billing, dashboard, multi-tenant hardening, possibly own
  number provisioning

## Open questions to resolve before any code

These are explicitly for us to discuss; the research can frame but not
decide.

1. **License for the SDK?**
   - **Lean:** MIT for the SDK; AGPL or BSL for the hosted runtime if
     we build one (open-core split).

2. **SMS provider for Phase 1?**
   - **Lean:** Telnyx (cost + DX + own network). Twilio adapter as
     Phase 1.5 for credibility.

3. **WhatsApp path: unofficial-only at first, or both?**
   - **Lean:** Baileys (unofficial) only for Phase 2. Cloud API in
     Phase 5.
   - Why: faster to demo, faster to feel-good. Cloud API onboarding
     is slow and dampens the "wow."

4. **WhatsApp unofficial scope — default-on or warning-gated?**
   - **Lean:** Available but with prominent risk disclaimers. Honest
     is the right move and good marketing.

5. **Brand & positioning?**
   - **Lean:** "Open-source messaging infrastructure for developers"
     with a punchier sub-tag once we find one. "Resend for SMS +
     WhatsApp" captures the energy but limits us.

6. **Target user — agencies, developers, or both?**
   - **Lean:** Developers Phase 1–4. Agencies become Phase 7 motion if
     traction supports.

7. **What's the one viral demo?**
   - Options:
     - "ChatGPT bot reachable via WhatsApp AND SMS in 20 lines"
     - "Self-hosted unified inbox: SMS + WhatsApp in one React UI"
     - "Smart routing: same `send()` call, automatically picks
       cheapest channel — animated GIF of the demo dashboard"
   - The smart-routing demo is the most novel. Lean toward it.

8. **Hosting target for our dev / demo runtime?**
   - Local Docker compose for dev
   - For demos: Fly.io, Railway, Render (persistent Node process,
     WebSocket support)
   - Not Vercel / CF Workers (long-lived WS, persistent session state)

9. **Should we ship SMS provider connections OR our own number
   provisioning?**
   - **Lean:** Pass-through (use customer's existing Twilio / Telnyx
     API keys) for Phase 1–4. Own provisioning is a Phase 7 hosted-
     product thing.

10. **How explicit about the "we are NOT for bulk marketing" stance?**
    - **Lean:** Very explicit. Front of README, blog post, demo copy.
      Distinguishes us, protects us legally, attracts the right
      developers.

## Why we should be honest about what could kill this

- **Twilio / Bird ship a great unified Node SDK.** Half our value prop
  vanishes.
- **Novu / Knock pivot to conversational.** They have momentum.
- **Evolution API ships English docs and an SDK.** They have LATAM
  momentum.
- **Meta launches a great official Node SDK for Cloud API.** Half our
  WhatsApp value prop vanishes.
- **Baileys changes license.** Foundation problem.
- **WhatsApp removes Web protocol entirely.** Unofficial path dies; only
  Cloud API survives.
- **A2P 10DLC tightens further.** SMS audience shrinks.
- **We get bored.** This is a long game (12+ months to traction). Make
  sure the picks above are ones we'd actually enjoy maintaining.

## What to read after this

When ready to commit, the next research artefact should be
`decision.md` — a one-page lock of:

- Chosen SMS provider for Phase 1
- Chosen WhatsApp driver for Phase 2
- Chosen license
- Chosen positioning
- Chosen Phase 0 scope (concrete file list)

After that we're in implementation mode and
`06-architecture-and-sessions.md` + `05-multi-channel-strategy.md`
become the blueprint.

## TL;DR (for future-us in a hurry)

> Multi-channel SDK (SMS + WhatsApp), unified API with smart routing.
> Telnyx for SMS + Baileys for WhatsApp first. MIT-licensed. Ship SDK
> first as OSS. Phase 1 = SMS only end-to-end, Phase 2 = add WhatsApp,
> Phase 3 = router, Phase 4 = Show HN. The wedge is DX + unification.
> The competitor to watch is Twilio Conversations (clunky) and Bird
> (enterprise-gated). Adjacent comparator is Novu (notifications, not
> conversations).
