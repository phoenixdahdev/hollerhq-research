# Library Comparison — Baileys vs OpenWA vs whatsapp-web.js

If we choose the unofficial path (or unofficial-as-one-mode), we have three
real options. They differ enough that the choice shapes the architecture.

## At a glance

| | Baileys | OpenWA (wa-automate) | whatsapp-web.js |
|---|---|---|---|
| npm | `@whiskeysockets/baileys` | `@open-wa/wa-automate` | `whatsapp-web.js` |
| Repo | `WhiskeySockets/Baileys` | `open-wa/wa-automate-nodejs` | `pedroslopez/whatsapp-web.js` |
| License | MIT | Mixed / key-gated | Apache-2.0 |
| Approach | Direct WebSocket | Puppeteer + JS injection | Puppeteer + JS injection |
| Per-session memory | ~30–80 MB | ~200–400 MB | ~200–400 MB |
| Per-session CPU at idle | Negligible | Moderate (Chrome) | Moderate (Chrome) |
| Cold start | ~1 s | ~10–20 s (browser launch) | ~10–20 s |
| Session storage | Small JSON (auth keys) | Chromium userDataDir (~50 MB) | Chromium userDataDir |
| Maintenance cadence | High | Medium (slowed) | High |
| TypeScript-native | Yes | Yes | Partial |
| Group features | Full | Full | Full |
| Status posting | Yes | Yes | Yes |
| Voice / video calls | Receive / decline only | No | No |
| Community size (2026) | Largest | Medium | Large |

## Baileys deep dive

**What it is:** A pure-Node implementation of WhatsApp's Signal-based
multi-device protocol. It opens a WebSocket to `wss://web.whatsapp.com/ws/chat`,
performs the auth handshake (or imports an existing one), and from there
exchanges binary Signal Protocol messages directly with WhatsApp's servers.

**Why it's the default choice now:**
- No browser → 5–10× less RAM per session
- No browser → no Chromium update breakage
- Faster — no IPC round-trips between Node and a browser tab
- MIT licensed cleanly
- Active community, many production deployments
- Sessions serialise to small JSON blobs — easy to back up, migrate,
  encrypt

**Where Baileys hurts:**
- You're reimplementing WhatsApp's protocol. When they change something
  subtle (key rotation, message format), Baileys needs a patch. The lag
  can be hours to days.
- Some media handling (specific WebP sticker variants, certain video
  codecs) is fiddly.
- Documentation is functional but not polished. Real understanding
  requires reading the source.
- Multi-device pairing flow involves QR or phone-number pairing — the
  API is fine, but error handling around expired / failed pairings is
  something you build yourself.

**Production pattern:** One Node process can host many sessions. People
run 50–500 sessions per process depending on activity. Beyond that you
shard across processes.

**Auth state model:**
```ts
// Baileys ships file-backed auth out of the box
import { useMultiFileAuthState, makeWASocket } from '@whiskeysockets/baileys';

const { state, saveCreds } = await useMultiFileAuthState('./sessions/customer-A');
const sock = makeWASocket({ auth: state });
sock.ev.on('creds.update', saveCreds);
```

For multi-tenant you replace `useMultiFileAuthState` with a custom adapter
backed by your DB. See `04-architecture-and-sessions.md` for the shape.

## OpenWA (wa-automate-nodejs) deep dive

Covered exhaustively in `01-openwa-deep-dive.md`. Short version: works,
more resource-heavy, license is a gotcha, community has drifted to
Baileys.

## whatsapp-web.js deep dive

**What it is:** A library similar in approach to OpenWA (Puppeteer +
injection of internal WhatsApp Web functions) but with a more
conventional API and clear Apache-2.0 license.

**Why people pick it over OpenWA:**
- Clean license
- Cleaner, more idiomatic API
- More active maintenance in recent years
- Larger English-language community on GitHub / Discord

**Why people pick it over Baileys:**
- Easier mental model if you understand WhatsApp Web
- Some edge cases (specific media types, business features) work more
  reliably out of the box
- More "high-level" — fewer protocol concepts to learn

**Why people don't:**
- Same resource cost as OpenWA — it's still Chromium

## Other contenders worth knowing

- **`@adiwajshing/baileys`** — old name for Baileys. Don't use; the fork
  is dead. Use `@whiskeysockets/baileys`.
- **`venom-bot`** — Brazilian community fork of OpenWA-style approach.
  Active there, less so internationally.
- **`whatsapp-web-multi-device-js`** — niche, lower adoption.
- **`wppconnect`** — next-most-popular Puppeteer-based option after
  whatsapp-web.js, especially in LATAM. Active community.

## Picking for our project

If the unofficial path is in scope at all, **default to Baileys**. The
resource math alone is decisive:

- 100 active sessions on a $40/mo VPS: feasible with Baileys, infeasible
  with Puppeteer-based libs
- 1000 active sessions across a small cluster: routine with Baileys,
  hard with Puppeteer

The only reasons to pick Puppeteer-based:
- We need a feature only exposed via WhatsApp Web's UI (rare)
- We want behavioural realism for ban avoidance (debatable benefit)
- Team has Puppeteer expertise and zero protocol expertise

For an SDK we plan to publish, Baileys also gives us a smaller dependency
footprint for consumers.

**Sub-recommendation:** if we want to abstract over both library families
for portability, design the internal `WhatsAppDriver` interface around
Baileys's capability model — it's the more constrained one in some media
areas, so building up is easier than building down.

## Cloud API client libraries — separate category

The libraries above are all for the **unofficial** path. If we're using
the official **WhatsApp Cloud API** (Phase 5 in the recommended path),
that's a different problem: HTTP REST client, not a long-lived
WebSocket.

Top options:

| Library | npm | License | Notes |
|---|---|---|---|
| **`@kapso/whatsapp-cloud-api`** | `@kapso/whatsapp-cloud-api` | MIT | Hand-crafted TS, Zod-validated builders, works standalone with Meta credentials. Kapso-proxy features are optional. **Strongest fit.** See `ecosystem/01-kapso.md`. |
| **`whatsapp-api-js`** | `whatsapp-api-js` | MIT | Older, single-maintainer, narrower surface |
| **Meta's official examples** | — | — | Raw `fetch()` snippets. Boilerplate-heavy. |
| **Twilio SDK (for WA)** | `twilio` | MIT | If routing via Twilio's BSP — different shape entirely |

**Recommendation for Phase 5:** wrap `@kapso/whatsapp-cloud-api` as our
Cloud API driver behind the `MessageDriver` interface. Saves ~1–2 weeks
of work building the client from scratch. License is clean (MIT,
confirmed in `package.json`); functionality covers what we need
(messages, templates, media, webhooks with HMAC verification, phone
numbers, flows). Detailed assessment in `ecosystem/01-kapso.md`.

## How to peek

A side-by-side memory test would settle this empirically. Rough plan:

```bash
# In a scratch dir, spin up 5 sessions of each, measure RSS
docker stats baileys-test wajs-test openwa-test
```

Expect to see:
- Baileys process: ~150–400 MB total for 5 sessions
- whatsapp-web.js / OpenWA: ~1.5 GB total for 5 sessions (one Chromium each)

If we ever want a number to put in marketing copy ("100 sessions in 4 GB"),
running this benchmark is how we get it honestly.
