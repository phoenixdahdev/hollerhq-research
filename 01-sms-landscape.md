# SMS Landscape

SMS feels old, but it's the most-reachable channel in existence — 6+
billion phones, no app install, regulated and tracked, used for OTP,
alerts, marketing, and customer support globally. Sending SMS
programmatically is also a deeply commoditised business with a
regulatory layer most developers underestimate.

## Why SMS still matters in 2026

- **Universal reach.** Every phone gets SMS. WhatsApp covers ~3B users;
  SMS covers ~6B.
- **OTP and transactional.** Banks, login flows, two-factor auth — SMS is
  still the de-facto default because regulators trust it.
- **Carrier-routed = high deliverability.** No spam folder (mostly).
  Aggressive filtering exists but a properly-configured SMS reaches its
  destination.
- **Audit trail.** Carriers log delivery; this matters for compliance.
- **Synchronous-enough.** Median delivery time: seconds.

**Where SMS loses to WhatsApp:**

- Per-message cost ($0.004–$0.05 vs $0–$0.005 typical WhatsApp service conv)
- No rich media reliably (MMS is US-only and unreliable internationally)
- 160-char segments, multi-part for longer
- No read receipts, no typing indicators
- One-way feel for most users (people don't reply to SMS like they
  reply to WhatsApp)

The two channels solve different problems. That's the whole basis for
going multi-channel — see `05-multi-channel-strategy.md`.

## Provider landscape

| Provider | Strength | Weakness | Price (US, approx) | DX |
|---|---|---|---|---|
| **Twilio** | Market leader, comprehensive | Expensive, sprawling product surface | $0.0079/msg out, $0.0075 in | Good but legacy-feeling |
| **Telnyx** | Owns its own carrier network, cheap | Smaller catalog | $0.004/msg out | Solid, dev-focused |
| **Plivo** | Twilio competitor, cheaper | Smaller community | $0.0055/msg out | OK |
| **Vonage (Nexmo)** | Enterprise, broad | Heavyweight | $0.0075/msg out | OK |
| **Bandwidth** | Own US carrier network | US-only focus | $0.004/msg out | Decent |
| **Sinch** | Global aggregator (acquired CLX, Inteliquent) | Sales-heavy | Enterprise quote | Mixed |
| **Infobip** | European, omnichannel | Enterprise-only feel | Enterprise quote | OK |
| **MessageMedia** | APAC strength | Regional | Regional | OK |
| **TextMagic** | SMB-focused | Small scale | $0.04+/msg | Simple |

### What "owning the carrier network" means

Twilio, Telnyx, and Bandwidth are different beasts from Plivo and Vonage.
Telnyx and Bandwidth own carrier interconnect agreements directly — they
don't buy capacity from upstream wholesalers. That's why their per-message
price is lower: no middleman margin. Twilio buys some capacity but also
operates direct interconnects in major markets.

For an OSS / SDK project we don't need to be in this business. We sit on
top of providers. But the choice of provider affects what we pass through
to users.

## Regulatory layer — where developers get blindsided

### US: A2P 10DLC (mandatory since 2023)

If you send "Application-to-Person" (A2P) messages from a regular 10-digit
long code (i.e., a normal phone number) to US recipients, you must
register:

1. **Brand** — your company, with EIN and verification (~$4–$40 one-time
   fee via The Campaign Registry / TCR)
2. **Campaign** — what you'll send (~$10/mo per campaign + per-msg fees)
3. **Number assignment** — link numbers to campaigns

Without registration: messages are heavily filtered or blocked outright.
With registration: throughput limits scale (T-Mobile: 75 msg/sec on
standard T-Mobile Verified Throughput).

This adds 2–4 weeks of onboarding for new US senders. Providers will
charge extra per-message fees on top of base SMS rate (Twilio's "carrier
fees" for A2P).

### US: Alternative channels

- **Toll-Free numbers** (1-8XX prefix): faster verification (1–3 weeks),
  can send A2P without 10DLC registration but still need toll-free
  verification.
- **Short codes** (5–6 digits): $500–$1500 / month, instant on-approval,
  high throughput (100+ msg/sec). Used for big consumer brands.

### US: TCPA — the legal landmine

The Telephone Consumer Protection Act. Civil liability of $500–$1500
per unconsented commercial message. Class-action territory. Lawsuits are
common; the plaintiff bar is well-organised.

Implications:

- Need documented consent before sending marketing SMS
- Need clear opt-out (STOP keyword handling — mandatory)
- Need to keep records (consent capture, opt-out timestamps)
- OTP / 2FA messages generally exempt (user-initiated)
- Established business relationship allows narrow transactional cases

We should bake this into our SDK: STOP keyword handling, consent tracking
hooks, opt-out lists checked on every send.

### EU + UK

- **GDPR** consent required for marketing SMS (Art. 6 lawful basis)
- **PECR** (UK) — soft opt-in allowed in narrow cases
- Right-to-be-forgotten applies to stored message contents

Implications:

- Don't store message contents longer than necessary
- Surface opt-out cleanly
- DPA needed if hosted

### Other notable regimes

- **India**: TRAI registration (DLT — Distributed Ledger Technology)
  required for commercial SMS — similar in spirit to 10DLC.
- **Brazil**: LGPD (GDPR equivalent).
- **Canada**: CASL (anti-spam) — strict consent, penalties up to $10M.
- **Australia**: Spam Act 2003.

## Sender ID types

Three distinct things:

1. **Long code**: normal 10-digit phone number. Bidirectional. Throughput
   limited (1 msg/sec base in US unregistered).
2. **Short code**: 5–6 digits. High throughput. US only, practically.
   Expensive.
3. **Alphanumeric sender ID**: company name as sender (e.g., "ACME").
   Available in many countries but **not US or Canada**.

A serious global SMS product surfaces all three; many startups only ship
long codes and discover later that EU customers want alphanumeric.

## Capabilities (the actually-usable surface)

| Capability | Reality |
|---|---|
| Plain text | ✓ (160 chars / segment, GSM-7 or UCS-2 encoding) |
| MMS (image / audio / video) | US-only mostly, expensive, often filtered |
| Long messages | ✓ (multi-part, billed per segment) |
| Delivery receipts | ✓ via webhook |
| Inbound (replies) | ✓ if long code or shared short code |
| Read receipts | ✗ (not in SMS) |
| Typing indicators | ✗ |
| Two-way threading | App-level, not protocol-level |
| RCS (Rich Communication Services) | Limited adoption; iPhone added support in iOS 18; carrier-specific; worth tracking, not foundational |

## Pricing realities

US-only consumer-facing app sending 100k msgs / month to long codes:

| Provider | Per-msg | A2P carrier fee | Monthly total (rough) |
|---|---|---|---|
| Twilio | $0.0079 | + $0.003 (T-Mobile) | $1,090 |
| Telnyx | $0.004 | + $0.003 | $700 |
| Plivo | $0.0055 | + $0.003 | $850 |
| Bandwidth | $0.004 | + $0.003 | $700 |

Twilio's premium pays for reliability, support, and being the safe
enterprise choice. Telnyx and Bandwidth win on cost if you can self-serve.

## The DX gap

What SMS APIs look like today:

```
POST /Accounts/{AccountSid}/Messages.json
  Authorization: Basic ...
  To=+15558675309&From=+15551234567&Body=Hello
```

What they could look like:

```ts
await messaging.sms.send({ to: '+15558675309', body: 'Hello' });
```

The gap isn't huge for one provider. The gap is huge when you start
needing provider-agnostic code, smart retries, channel fallback to
WhatsApp, structured templates, or local development without burning
credits.

**Concrete pain points across existing SMS SDKs:**

- **Twilio**: types are auto-generated from OpenAPI — they exist but
  don't feel hand-crafted. SDK is huge (covers all Twilio products).
  Errors are Twilio-shaped, not domain-shaped.
- **Vonage**: TypeScript types added recently, still rough.
- **Plivo**: SDK is fine but generic.
- **Telnyx**: SDK works, docs sparse compared to Twilio.

**What "good SMS DX" looks like:**

- Strict, hand-crafted TS types
- One-line `send()` for the common case
- Built-in STOP / opt-out handling
- Built-in idempotency (retry without duplicate send)
- Webhook helpers with HMAC verification
- Local dev mode with a mock that records and replays
- Clear provider-level error mapping (insufficient credits, invalid
  number, unsubscribed, carrier-blocked)
- Cost estimation API (estimate cost before send)

## How this informs our project

Three real shapes:

1. **Wrap one provider** (e.g., Telnyx). Smaller surface, opinionated,
   tightly coupled.
2. **Multi-provider adapter** (`SMSProvider` interface, ship adapters
   for Twilio + Telnyx at minimum). More work, but matches the
   "developer freedom" story.
3. **Hosted by us** — we provision numbers and pass through. Means
   dealing with 10DLC ourselves; biggest operational burden.

**Lean:** start with Option 2. Default driver = Telnyx (cost + DX);
ship Twilio adapter early because nobody trusts a new SMS SDK that
doesn't support Twilio. See `05-multi-channel-strategy.md` for how the
SMS provider interface interacts with the WhatsApp driver interface.

## How to peek

- Sign up for Twilio (free $15 credit), send yourself an SMS in 10
  minutes.
- Sign up for Telnyx (free credit), do the same. Compare onboarding.
- Skim the TCR registration flow if you want to feel the 10DLC pain
  ahead of time: [www.campaignregistry.com](https://www.campaignregistry.com/).
- Read Twilio's [Programmable Messaging docs](https://www.twilio.com/docs/messaging).
  Note how the SDK feels and what's hard to find.
