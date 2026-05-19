# Competitive Landscape (SMS + WhatsApp + Multi-Channel)

Knowing who's already shipping is the first step to seeing where the
gap is. The market has overlapping but distinct layers: SMS providers,
WhatsApp tools, multi-channel platforms that try to unify both, and
adjacent notification infrastructure.

## Layer 1: SMS providers (infrastructure we sit on)

| Provider | Position vs us |
|---|---|
| **Twilio** | The Goliath. Comprehensive but expensive. SDK is auto-generated; DX is OK not great. We adapter-wrap and provide better DX. |
| **Telnyx** | Cheaper than Twilio, owns own network. Solid SDK but no multi-channel story. We sit on top. |
| **Plivo** | Cheaper Twilio clone. Smaller community. We sit on top. |
| **Vonage** | Enterprise sales-led. Heavy. We sit on top. |
| **Bandwidth** | Own US carrier network. Mid-market US focus. We sit on top. |
| **Sinch** | Aggregator (acquired CLX, Inteliquent). Enterprise. We sit on top. |
| **Infobip** | European omnichannel enterprise. We compete on DX, not features. |

These aren't competitors at the SDK layer — they're underlying
infrastructure. We wrap them. We could compete with **Twilio's
Programmable Messaging SDK** specifically on developer experience.

## Layer 2: WhatsApp OSS libraries (foundations AND competitors)

| Project | Stars (approx) | Position vs us |
|---|---|---|
| **Baileys** (`@whiskeysockets/baileys`) | ~14k | Our likely WhatsApp foundation |
| **OpenWA** (`open-wa/wa-automate-nodejs`) | ~3.5k | Possible foundation; license concerns |
| **whatsapp-web.js** | ~16k | Foundation alternative |
| **wppconnect** | ~7k | LATAM-popular |
| **venom-bot** | ~6k | Brazilian community |

Libraries, not products. None ships a hosted runtime, multi-tenant
management, dashboards, billing, or webhook delivery.

## Layer 3: WhatsApp-only hosted services

| Service | Approach | Pricing | Position |
|---|---|---|---|
| **Wassenger** | Unofficial-based hosted | ~$30+/mo per number | Polished, established |
| **Chat-API.com** | Unofficial-based | ~$20+/mo | Established, simple API |
| **Green-API** | Unofficial-based | Pay-per-message | Big in CIS markets |
| **Maytapi** | Unofficial-based | $50+/mo | Israel-based |
| **Whapi.cloud** | Unofficial-based | $39+/mo | EU-friendly |
| **Ultramsg** | Unofficial-based | $39+/mo | MENA |
| **Wati** | Cloud API + UI | $39+/mo + Meta | SMB focus |
| **360Dialog** | Cloud API BSP | Per-message | EU enterprise |
| **Gupshup** | BSP omnichannel | Volume-based | India-first |
| **Twilio for WhatsApp** | Cloud API reseller | Pass-through + markup | Enterprise default |
| **Kapso** (kapso.ai) | Cloud API + developer platform | Tiered (paid SaaS + OSS funnel) | "WhatsApp for developers" — closest direct DX competitor. WhatsApp-only, no unofficial / SMS / multi-channel. Deep dive: `ecosystem/01-kapso.md`. Their `@kapso/whatsapp-cloud-api` (MIT) is our planned Cloud API driver foundation. |

WhatsApp-only space is crowded but everyone is selling hosting + UI,
not SDK. None open source. None with a polished SDK.

## Layer 4: WhatsApp-related OSS server projects

| Project | Stack | Position vs us |
|---|---|---|
| **Evolution API** | Node, REST | Closest OSS competitor. REST on top of Baileys. LATAM strong. No SDK. Thin English docs. |
| **WPPConnect Server** | Node, REST | REST on top of wppconnect. Sparse docs. |
| **Chatwoot** | Rails | Multi-channel inbox UI. Cloud-API-only for WA. Inbox-focused not SDK-focused. |
| **whatsapp-api-js** | TS | Cloud API TS wrapper. Single-channel. |

**Evolution API is the closest existential threat.** Same idea but no
English docs, no SDK polish. We can be the developer-product version
of this.

## Layer 5: Multi-channel commercial platforms (most direct competition)

Where the unified-SDK story plays out.

| Service | What they offer | DX | Gap |
|---|---|---|---|
| **Twilio Conversations** | Multi-channel inbox + API (SMS, WA, FB Messenger) | Heavy, multi-product, auto-gen SDK | DX |
| **Bird (MessageBird)** | Multi-channel SaaS | Enterprise sales | No OSS, no self-host |
| **Sinch** | Multi-channel global aggregator | Sales-led | No DX wedge |
| **Infobip** | Multi-channel enterprise omnichannel | Enterprise | No DX wedge |
| **Respond.io** | Multi-channel inbox | Inbox-focused | Not an SDK |
| **Sleekflow** | Multi-channel CRM | CRM-focused | Not an SDK |
| **Brevo (Sendinblue)** | Email + SMS + WA marketing | Marketing-focused | Different audience |

Multi-channel exists at the enterprise SaaS layer. None of these ship
a TS-first SDK that competes with Stripe / Resend on DX.

## Layer 6: Notification infrastructure (adjacent but different)

| Project | Channels | Position |
|---|---|---|
| **Novu** | Email, SMS, push, in-app, chat | OSS, well-funded, growing fast. One-way notifications focus. |
| **Knock** | Email, SMS, push, in-app, Slack, etc. | Hosted, similar to Novu. One-way. |
| **Courier** | Email, SMS, push, in-app, chat | Hosted, one-way notifications. |
| **MagicBell** | In-app + push | Narrow scope |

These are the **closest DX-quality comparisons** but they're
notifications (one-way, trigger-driven, idempotent), not conversations
(two-way, stateful, contextual). The product shape is different.

We can position by contrast:

> "Novu / Knock / Courier for sending notifications. Us for
> conversational messaging."

## Layer 7: Open-source adjacent (not competitors)

| Project | Role |
|---|---|
| **n8n** | Workflow automation. Could integrate our SDK as a node. |
| **Trigger.dev** | Background jobs. Similar OSS positioning, similar DX bar. Inspiration. |
| **Resend** | Email API. The DX role model for our space. |
| **Loops** | Email + transactional. Modern API, marketing-leaning. |

Not competitors — brand exemplars (Resend, Trigger.dev) or potential
integration targets (n8n, Chatwoot, Typebot).

## Where the gaps actually are

### Gap 1: Truly developer-grade SDK across SMS + WhatsApp

Nobody ships a Stripe-quality SDK that handles both channels with
smart routing. Twilio Conversations exists but the DX is mediocre.

### Gap 2: Open-source self-hosted runtime that's multi-channel

Chatwoot is multi-channel but inbox-focused. Evolution API is WA-only
and no SDK. Room for the "Supabase of conversational messaging" — OSS
+ optional cloud + great DX.

### Gap 3: Smart channel routing as a first-class feature

"Try WhatsApp first because it's free, fall back to SMS" is the
obvious optimization. Nobody does it as the default. Wati comes
closest but only inside their UI.

### Gap 4: A canonical Node.js multi-channel community

Library community is fragmented. A project that becomes the canonical
Node.js way to build conversational messaging — opinionated, well-
maintained — could become infrastructure.

## Adjacent inspirations (what "good" looks like)

- **Resend** (`resend.com`) — email API; ~10-line SDK calls; perfect
  docs; React Email integration. Bar to aim for.
- **Stripe** — canonical DX example.
- **Supabase** — OSS + optional cloud template.
- **Trigger.dev** — typed durable jobs, React-ish DX in Node.
- **Novu** — OSS notification infra. Good comparable for "OSS infra
  with optional cloud."

## What "getting seen" looks like in this market

Three plays that work:

1. **Pre-launch traction:** demos on Twitter / HN that show one-line
   API → working SMS + WhatsApp bot. Visual demos (smart-routing
   dashboard) do well.
2. **"Show HN" of a v0.1 SDK + hosted demo.** HN crowd has soft spot
   for "self-hosted Twilio alternative" and "reverse-engineered X"
   stories.
3. **Become the dependency.** If Chatwoot, n8n, or Typebot integrates
   our SDK, we're in.

The "road to getting seen" is consistent shipping + one viral demo +
one upstream integration.

## Honest competitive risks

- **Twilio / Bird ship a real unified Node SDK.** Today their SDKs are
  auto-generated and dull. If they get serious about DX, our wedge
  narrows.
- **Kapso extends to SMS or Baileys.** Today they're WhatsApp-Cloud-API-
  only. If they add SMS and an unofficial WA driver, they become a head-
  on competitor in our exact lane. Watch `github.com/gokapso` for new
  repos as a signal.
- **Novu / Knock add conversational features.** Momentum and
  infrastructure. If they pivot from notifications to conversations,
  they're well-positioned.
- **Evolution API ships English docs and an SDK.** LATAM momentum.
- **Meta launches an official Node SDK for Cloud API.** Would
  commoditize half our WhatsApp value prop.
- **Baileys changes license.** Foundation problem (cf. OpenWA).
- **WhatsApp removes Web protocol.** Unofficial path dies.
- **TCPA / 10DLC tightens.** SMS audience shrinks for small senders.

## Summary

| Layer | Our relationship |
|---|---|
| SMS providers (Twilio, Telnyx, etc.) | We adapt-over, don't compete |
| WhatsApp OSS libs (Baileys, OpenWA) | We adapt-over, don't compete |
| WhatsApp SaaS (Wassenger, Wati) | We compete on OSS + DX |
| WhatsApp OSS servers (Evolution API) | Direct competitor — DX is our wedge |
| Multi-channel SaaS (Twilio Conv, Bird) | We compete on OSS + DX |
| Notification infra (Novu, Knock) | Adjacent — we're conversational, they're one-way |
