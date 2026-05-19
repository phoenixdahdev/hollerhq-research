# Kapso — Analysis & Leverage Opportunities

## TL;DR

[Kapso](https://kapso.ai) ([github.com/gokapso](https://github.com/gokapso))
is a "WhatsApp for developers" company with a hosted SaaS plus ~9 OSS
repos. They're squarely on the **Cloud API** path (not unofficial /
Baileys). One of their repos — **`@kapso/whatsapp-cloud-api`** — is a
high-quality MIT-licensed TypeScript Cloud API client we should adopt
as our Cloud API driver instead of writing one ourselves. Most other
repos are either Kapso-platform-locked or have unclear licensing
(LICENSE file missing). They're now our closest direct DX competitor
on the WhatsApp-Cloud-API axis.

## What is Kapso?

- Company at [kapso.ai](https://kapso.ai), self-described as
  "WhatsApp for developers"
- 220 GitHub followers (research date — meaningful traction)
- Docs at [docs.kapso.ai](https://docs.kapso.ai)
- Cloud-API-only positioning (no unofficial / Baileys play)
- Most consumer-facing OSS repos require a Kapso account
- Hosted SaaS appears to be paid (pricing details JS-rendered, not
  cleanly crawlable; spike with an account to confirm)

## Repo-by-repo assessment

Verified via `https://api.github.com/orgs/gokapso/repos` plus
`package.json` and LICENSE inspection on `master`.

| Repo | Stars | License | Standalone? | Action |
|---|---|---|---|---|
| `whatsapp-cloud-inbox` | 665 | MIT | **No** — requires Kapso account | Inspiration only |
| `whatsapp-cloud-api-js` | 151 | **MIT** (package.json; no LICENSE file) | **Yes** | **Adopt** — wrap as Cloud API driver |
| `agent-skills` | 114 | None (no LICENSE, no license field) | Yes (but legally murky) | Pattern only — publish our own |
| `whatsapp-broadcasts-example` | 46 | MIT | No — uses Kapso API | Inspiration only |
| `claude-code-whatsapp` | 38 | None | No — needs Kapso | Idea inspiration |
| `whatsapp-support-agent` | 31 | MIT | Likely needs Kapso | Inspiration |
| `whatsapp-voice-agent-pipecat` | 21 | BSD-2 | Needs Kapso for WA | Bookmark for voice angle |
| `kapso-workflows` | 4 | None | No — Kapso-only | Irrelevant |
| `metabase` | 0 | MIT | Standalone (Render template) | Irrelevant |

## The find: `@kapso/whatsapp-cloud-api`

This is the leverage opportunity.

### Verified facts

- **License**: MIT (declared in `package.json` on `master` branch). No
  LICENSE file in repo — minor cleanliness issue but legally MIT via
  the manifest.
- **Version**: `0.2.1` (pre-1.0; expect API churn)
- **Node**: requires `>=20.19`
- **Stack**: TypeScript, Zod 4 for validation, tsdown for build,
  vitest for tests, msw for HTTP mocking
- **Exports**: `.` (client) and `./server` (server-side helpers,
  likely webhook signature verification)
- **Runtime deps**: just `zod` — clean dependency footprint

### What it covers

Per their README:

- **Messages**: text, media (image / video / document / sticker),
  locations, contacts, reactions, interactive buttons, CTAs, catalog
  messages, templates
- **Templates**: creation, retrieval, sending with typed builders
  (`buildTemplateSendPayload`, `buildTemplateDefinition`)
- **Media**: upload, metadata, deletion, download
- **Webhooks**: signature verification (HMAC) and event normalisation
  (camelCased fields)
- **Phone Numbers**: registration, verification, settings, business
  profile management
- **Flows**: authoring, validation, deployment, preview

Optional Kapso proxy adds: conversations, message-history queries,
contacts, call logs. These are gated behind `baseUrl + kapsoApiKey`
config — if you don't set those, the library talks directly to Meta.

### Why it's a strong fit

- **Hand-crafted TS types**, not auto-generated from OpenAPI. Exactly
  the DX bar we'd set ourselves.
- **Zod validation** at the boundary — runtime + compile-time safety.
- **Webhook helpers** with HMAC verification — table stakes we'd
  otherwise build.
- **Modular surface** — `client.messages.sendText`,
  `client.templates.create`, etc. Matches our intended driver
  interface shape.
- **Two-file build** (CJS + ESM) — works in both runtimes.
- **Minimal deps** — small footprint for our SDK consumers.

### Minimal usage (from their README)

```ts
import { WhatsAppClient } from '@kapso/whatsapp-cloud-api';

const client = new WhatsAppClient({
  accessToken: process.env.WHATSAPP_TOKEN!,
});

await client.messages.sendText({
  phoneNumberId: '<PHONE_NUMBER_ID>',
  to: '+15558675309',
  body: 'Hello!',
});
```

### How we'd wrap it

In our monorepo, `@your-org/whatsapp-driver-cloud-api`:

```ts
import { WhatsAppClient } from '@kapso/whatsapp-cloud-api';
import type { MessageDriver, SendRequest, SendResult } from '@your-org/messaging-core';

export class CloudApiWhatsAppDriver implements MessageDriver {
  channel = 'whatsapp' as const;
  capabilities = {
    media: true, buttons: true, templates: true,
    groups: false,                          // Cloud API limitation
    pricePerMessageUsd: undefined,          // depends on conversation type
  };

  private client: WhatsAppClient;
  private phoneNumberId: string;

  constructor(opts: { accessToken: string; phoneNumberId: string }) {
    this.client = new WhatsAppClient({ accessToken: opts.accessToken });
    this.phoneNumberId = opts.phoneNumberId;
  }

  async send(req: SendRequest): Promise<SendResult> {
    if (req.body.type === 'text') {
      const result = await this.client.messages.sendText({
        phoneNumberId: this.phoneNumberId,
        to: req.to,
        body: req.body.text,
      });
      return { external_id: result.messages[0].id, status: 'SENT' };
    }
    // ... handle media, template, interactive
  }

  // Optional session methods left undefined — Cloud API has no
  // session lifecycle (no QR, no pairing).
}
```

A few days of work to wrap properly vs ~2 weeks to write the Cloud API
client from scratch. Net saving: ~1.5 weeks.

### Risks of depending on it

- **Pre-1.0**: API will change. Pin version, upgrade deliberately.
- **Single-vendor maintenance**: if Kapso pivots or shuts down, we'd
  fork. MIT permits this; fork would live under our org.
- **Brand association**: customers will see `@kapso/*` in our
  dependency tree. Mitigation: re-export via our wrapper so consumers
  only see `@your-org/whatsapp-driver-cloud-api`.
- **No LICENSE file in repo**: package.json declaration is legally
  sufficient but worth documenting that we verified it.

### Validation spike (do this before committing)

```bash
mkdir scratch-kapso && cd scratch-kapso
npm init -y
npm install @kapso/whatsapp-cloud-api
```

Send a message from a Meta sandbox number. 30–60 minutes is enough to:

- Confirm API ergonomics match the README
- Check error-message quality
- Inspect webhook helpers (does HMAC verification really work?)
- Get a feel for the Zod schemas (strict enough?)

If the spike validates, commit to using it.

## Inspiration but don't adopt

### `whatsapp-cloud-inbox` (665 stars, MIT)

A Next.js inbox UI. **Requires a Kapso account** — coupling that makes
it unusable directly.

What to study:

- 24-hour-window enforcement UX
- Template builder UI (probably the best in OSS)
- Interactive-button send UX
- Auto-polling pattern for new messages (we'd use SSE / WebSocket, but
  worth seeing their approach)

Our dashboard does NOT fork this — we build our own with Drizzle +
Hono backend (not Kapso-coupled). But we steal the UX patterns.

### `whatsapp-broadcasts-example` (46 stars, MIT)

Next.js demo of mass messaging via Kapso's Broadcasts API. Same
coupling problem.

Study: how they present fanout UI (recipient list selection, template
selection, send progress). Our `fanoutBroadcast` workflow
(Trigger.dev, per `engine/01-requirements.md`) needs a UI on top —
this is a good reference.

### `whatsapp-voice-agent-pipecat` (21 stars, BSD-2)

WhatsApp voice agent using Kapso + Pipecat. Voice on WhatsApp is hard
and largely undocumented; this is the only OSS reference for it.

Bookmark for Phase 7+ if voice ever becomes a roadmap item.

## What we shouldn't depend on (unclear license)

`agent-skills`, `claude-code-whatsapp`, `kapso-workflows`,
`whatsapp-support-agent` (some have MIT in repo metadata but missing
LICENSE files — verify case by case).

**`agent-skills`** is the most interesting. It follows the open
[Agent Skills specification](https://agentskills.io/specification),
which is a vendor-neutral format for AI-agent capabilities. Three
skills: `integrate-whatsapp`, `automate-whatsapp`, `observe-whatsapp`.

**Recommendation:** publish our OWN agent skills following the open
spec. Mirror their pattern:

- `integrate-messaging` (connect SMS + WhatsApp, send first message)
- `automate-messaging` (build workflows on top of our SDK)
- `observe-messaging` (debug deliveries, check session health)

Becomes a distribution channel — AI agents (Claude, Cursor, etc.) can
discover and install our skills.

## Strategic / competitive implications

### Kapso is now a known direct competitor

Before this discovery, our competitive landscape doc named Evolution
API (OSS) and Twilio Conversations (commercial) as our closest
threats. Kapso changes that — they're a developer-first WhatsApp
platform with:

- Real momentum (665-star flagship, 220 followers, regular commits)
- A hosted product (kapso.ai SaaS)
- Strong DX (their library is the bar we'd otherwise set ourselves)
- An AI-agent integration story (agent-skills, support agent, voice
  agent, claude-code)

They are executing **half** of the playbook from
`research/09-recommended-path.md` — the WhatsApp half.

### What they leave open (our wedge)

- **No SMS.** Entirely WhatsApp-focused.
- **No unofficial / Baileys.** Cloud-API-only.
- **No multi-channel routing.** Single-channel architecture.
- **No general self-host runtime.** Their OSS is either client
  libraries or Kapso-coupled apps; you can't `docker compose up` to
  get the whole platform.

We can be Stripe-quality on the Cloud API path (by adopting their
library) AND ship multi-channel + unofficial + self-host where they
don't play.

### Positioning options

**Option A — direct head-on:**

> "Better DX than Kapso. Plus SMS. Plus Baileys. Plus self-host."

Honest but combative. Devs may take Kapso's side if they already use
them.

**Option B — complementary:**

> "Kapso is great for WhatsApp Cloud API. We're for the whole stack —
> SMS + WhatsApp (Cloud API and unofficial), unified, self-hostable."

Honest and credits them; better long-term relationship. Lean B.

### Risk: they extend to SMS

If Kapso adds an SMS channel and an unofficial WA driver, they become
a head-on competitor in our exact lane. Probability is non-zero —
adding a Twilio adapter to their existing platform is a few weeks of
work for them.

Mitigation: ship fast, build the community moat, make our Baileys
story deeper than they'd attempt as a Cloud-API-first company.

## Recommended action plan

### Now (research stage)

- [x] Verified `@kapso/whatsapp-cloud-api` is MIT licensed
- [x] Documented the leverage opportunity (this file)
- [ ] Run the 60-min validation spike with a Meta sandbox number
- [ ] Decide: adopt as-is, or wrap (lean: wrap for interface isolation)

### Phase 5 (Cloud API driver work)

- Wrap `@kapso/whatsapp-cloud-api` in
  `@your-org/whatsapp-driver-cloud-api`
- Pin to a specific version (don't auto-upgrade pre-1.0 library)
- Re-export types we expose; hide Kapso-specific types from our public
  API
- Document Kapso credit in our README's "Acknowledgements"

### Phase 4+ (marketing / positioning)

- Acknowledge Kapso in launch docs (Option B positioning)
- Publish open Agent Skills mirroring their three (integrate /
  automate / observe — for our SDK)
- Bookmark `whatsapp-cloud-inbox` UX patterns for our dashboard build

### Phase 7+ (optional)

- If voice ever becomes a roadmap item, study
  `whatsapp-voice-agent-pipecat` as the reference

## Open questions

1. **Is the API stable enough to bet on?** v0.2.1 is pre-1.0.
   Mitigation: pin version, fork if abandoned.
2. **Will Kapso react negatively to us using their library at scale?**
   Unlikely — MIT is permissive. Possibly positively
   (cross-pollination).
3. **Should we contribute back?** Yes if we find / fix bugs. Builds
   goodwill and reduces fork debt.
4. **Should we reach out to Kapso directly?** Optional. A cordial
   "we're building X, using Y of yours, here's how we differ" note
   could be useful. Or wait until we ship something to talk about.
5. **Will they extend to SMS?** Watch their GitHub closely. Any
   sms-related repo appearing in `gokapso/` is a signal to react.

## How to peek

Already documented above — install `@kapso/whatsapp-cloud-api`, get a
Meta sandbox number, send a text in 30 minutes. The Zod schema and
typed methods will tell you immediately whether the DX is what we
want.

## Sources

- [github.com/gokapso](https://github.com/gokapso) — organization page
- [github.com/gokapso/whatsapp-cloud-api-js](https://github.com/gokapso/whatsapp-cloud-api-js) — the library
- `package.json` on `master` branch — license verification (MIT)
- [kapso.ai](https://kapso.ai) — company landing page
- [docs.kapso.ai](https://docs.kapso.ai) — documentation site
- [agentskills.io/specification](https://agentskills.io/specification) —
  open spec their agent-skills repo follows
