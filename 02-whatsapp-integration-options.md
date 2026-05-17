# WhatsApp Integration Options — The Landscape

WhatsApp has no single "API." There are three categorically different paths,
each with its own audience, cost structure, capability profile, and risk
profile. The choice between them is the single most important decision in
this project.

## Path A: WhatsApp Cloud API (official, Meta-hosted)

The official path that Meta launched in May 2022. Replaced the older
On-Premises API for new customers; existing On-Premises customers are being
migrated.

**How it works:**
- Create a Meta Developer account and a WhatsApp Business Account (WABA)
- Verify your business with Meta (Business Verification — takes days,
  requires docs)
- Meta provisions a phone number (or you bring your own)
- You get a permanent access token and use the Cloud API at
  `https://graph.facebook.com/v17.0/{phone-id}/messages`
- Meta runs the actual WhatsApp infrastructure for you
- You receive incoming messages via webhooks

**Pricing model:**
Meta uses "conversations" (24-hour message threads), not per-message billing.

- **Service conversations**: $0 (user-initiated, since Nov 2024 update)
- **Marketing conversations**: $0.008–$0.15 per conversation depending on
  country
- **Utility conversations**: $0.004–$0.05
- **Authentication conversations**: $0.014–$0.075

So a single 50-message back-and-forth in a 24-hour service window costs $0.

**Capabilities:**
- Send text, media (image / video / audio / document), templates,
  interactive messages (buttons, lists), reactions
- Receive everything
- Read / typing indicators
- Group management: **No** — Cloud API does not support groups
- Status (story) posting: **No**
- One-on-one only

**Constraints that bite:**
- **Template messages required** for business-initiated conversations.
  Templates must be pre-approved by Meta — turnaround can be hours to days.
- **24-hour customer service window.** After a user messages you, you have
  24 hours to reply freely. After that, only pre-approved templates.
- **Quality rating system.** Your number gets a quality rating; if users
  block / report you, rating drops and Meta throttles you.
- **Tiered messaging limits.** New numbers start at 250 unique recipients /
  day; you graduate to 1k → 10k → 100k → unlimited based on quality and
  volume.
- **Business verification.** Non-trivial; blocks individual developers
  casually trying things.

**When to use:**
- Building for legitimate businesses
- Compliance and reliability matter more than feature richness
- OK with templates and 24-hour windows
- Don't need groups

**Pros:**
- Sanctioned, zero ban risk
- Meta runs the infrastructure
- Generous free tier for service conversations
- Templates and interactive messages are first-class

**Cons:**
- Slow setup (business verification)
- Template approval friction
- No group features
- You're a tenant in Meta's ecosystem — they can change rules at any time

## Path B: Business Solution Providers (BSPs)

Resellers / integrators on top of the Cloud API or on top of legacy
On-Premises infrastructure. They abstract the friction of getting onboarded
with Meta and add their own UI / features.

**Examples:**
- **Twilio for WhatsApp** — pay Twilio per message + Meta's conversation
  fees. Easy onboarding via Twilio sandbox.
- **360Dialog** — European BSP, popular with EU companies
- **Gupshup** — India-focused, full omnichannel
- **MessageBird (now Bird)** — omnichannel
- **Wati** — SMB-focused with a UI on top of Cloud API
- **Respond.io** — multi-channel inbox
- **Vonage** — enterprise

**How they differ from raw Cloud API:**
- Faster onboarding (some have sandbox numbers for instant testing)
- Markup on per-message price (usually $0.005–$0.02 added per conversation)
- Value-add: SDKs, dashboards, inbox UIs, analytics
- Some still run On-Premises for clients who haven't migrated

**When to use:**
- Want speed-to-launch and don't mind paying margin
- Need a vendor with SLAs
- Don't want to deal with Meta directly

**When they're irrelevant to this project:**
- We're building an SDK / platform, not consuming one — BSPs are our
  competition or our target user (depending on how we position).

## Path C: Unofficial libraries (WhatsApp Web automation)

What OpenWA is. You impersonate a WhatsApp Web client and drive it
programmatically. No Meta involvement, no fees, no templates, no 24-hour
windows, no verification — and no sanction.

**Sub-categories:**

- **Browser-based** (Puppeteer + JS injection): OpenWA, whatsapp-web.js,
  wppconnect, venom
  - Heavier, slower, more memory
  - Closer to "real" browser behaviour (arguably better for ban avoidance —
    debated)
- **Protocol-based** (raw WebSocket): Baileys
  - Lightweight, fast, scalable
  - More direct — easier to fingerprint as automation *in theory*; in
    *practice* Baileys is widely deployed without immediate bans, suggesting
    Meta is more focused on behavioural signals than fingerprinting

**Capabilities (all sub-categories):**
- Everything WhatsApp Web can do: groups, status, reactions, polls, business
  catalogs, etc.
- Send to any number that has WhatsApp — no opt-in required by the protocol
  (though you create legal / ToS exposure)
- No per-message cost
- No template approval

**Costs:**
- Free in the literal sense
- Real cost is operational: hosting, session management, ban-handling, ToS
  risk

**When to use:**
- You need features Cloud API doesn't have (groups, status, full media)
- You can't afford per-message pricing at your volume
- You're explicitly OK with the gray area (and your users are too)
- The user base you serve uses WhatsApp in ways the official API doesn't
  fit (community moderators, group automation, personal assistants)

**When to avoid:**
- Building for regulated industries (banking, healthcare)
- Building anything bulk-marketing-shaped — bans come fast
- Users would be harmed by sudden account bans

## Decision matrix

| Concern | Cloud API | BSP | Unofficial |
|---|---|---|---|
| Setup time | Days–weeks | Hours–days | Minutes |
| Per-message cost | Meta's fees | Meta + markup | $0 |
| Groups support | No | No | Yes |
| Status posting | No | No | Yes |
| Bulk messaging | Restricted (quality tier) | Same | Risky (bans) |
| Sanctioned | Yes | Yes | No |
| Ban risk | None | None | Real |
| Infrastructure burden | Low | Lowest | High (sessions) |
| Audience | Businesses | Businesses | Tinkerers, devs, agencies |

## What this means for us

The three paths target genuinely different audiences. We need to pick one
as our **first wedge**:

1. **If we go Cloud API:** We're competing with Twilio and BSPs on DX. The
   market is crowded but lucrative. We need a sharp DX wedge (e.g., "the
   Stripe of WhatsApp Cloud API" — perfect docs, typed SDKs, local
   emulator).

2. **If we go Unofficial (Baileys / OpenWA-based):** We're competing with
   Wassenger, Chat-API, etc. Less crowded, faster traction with developers,
   but ban risk is real and onus is on us to manage it. Open-source wedge
   works well here because the community already lives there.

3. **If we go Both (abstraction over both):** Most ambitious. "Use Cloud
   API where it works, fall back to Web automation for groups / etc." A
   compelling story, but we inherit both sets of problems.

Recommendation lives in `07-recommended-path.md`. Detailed competitor view
in `06-competitive-landscape.md`.

## How to peek / sanity-check

- **Cloud API**: read [developers.facebook.com/docs/whatsapp/cloud-api](https://developers.facebook.com/docs/whatsapp/cloud-api).
  Notice the "First call" flow — getting a test sandbox number takes ~15
  minutes; getting a production WABA verified takes days.
- **Twilio's WhatsApp sandbox**: signup → join sandbox by texting a code →
  send messages within ~10 minutes. Good for feel of the BSP experience.
- **Baileys**: clone [WhiskeySockets/Baileys](https://github.com/WhiskeySockets/Baileys),
  run their example, scan QR. Twenty minutes start-to-message. The contrast
  in setup time with Cloud API is itself instructive.
