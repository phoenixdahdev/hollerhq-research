# OpenWA — Deep Dive

## What it actually is

The repo you found — [rmyndharis/OpenWA](https://github.com/rmyndharis/OpenWA) —
is a fork. The upstream is **[open-wa/wa-automate-nodejs](https://github.com/open-wa/wa-automate-nodejs)**,
typically referenced as **OpenWA** or **wa-automate**. Before going deeper we
should work from the upstream because the fork may be outdated or modified;
the API reference and changelog only track upstream.

OpenWA is a Node.js library that controls WhatsApp Web through a headless
browser (Puppeteer / Chromium). It injects JavaScript into the WhatsApp Web
page to call internal WhatsApp store functions directly — meaning it doesn't
just simulate clicks, it reaches into the in-page application state and
invokes the same functions WhatsApp's own UI uses.

This is meaningful because:

- **It's more reliable than UI automation.** You're calling JS functions, not
  clicking buttons that may move.
- **It's also more fragile than a clean protocol implementation.** When
  WhatsApp ships a new Web build that renames internal functions, the library
  breaks until someone patches the function map.
- **It's reverse-engineered.** Every release of WhatsApp Web is a potential
  break.

## How it works (mechanical view)

```
your-node-process
        │
        ▼
  Puppeteer launches Chromium
        │
        ▼
  Navigates to web.whatsapp.com
        │
        ▼
  Injects wapi.js (custom JS bundle that exposes
                   WA internals as window.WAPI.*)
        │
        ▼
  Your Node code calls client.sendMessage(...)
        │
        ▼
  Library bridges that to await page.evaluate(
    () => window.WAPI.sendMessage(...)
  )
        │
        ▼
  WhatsApp Web sends the message over its WebSocket
  to WA servers
```

The injected `wapi.js` is the heart of it. It hooks into WhatsApp Web's
Webpack modules at load time, finds and re-exports message-store functions,
contact functions, group functions, etc. When the Webpack chunk hashes
change (which happens on every WA release), the hook code needs to be
adjusted.

## Capabilities

| Feature | Supported |
|---|---|
| Send/receive text | Yes |
| Images, video, audio, documents, stickers | Yes |
| Locations, contacts (vCard) | Yes |
| Groups: create, add, remove, promote | Yes |
| Status (story) posting | Yes |
| Read receipts, typing indicators | Yes |
| Profile picture changes | Yes |
| Business catalogs / orders | Partial |
| Reactions, replies, mentions | Yes |
| Forwarded-message detection | Yes |
| Polls | Yes (recent versions) |
| Voice / video calls | **No** — not exposed via Web |

## Limitations

- **One Chromium per session.** Memory cost is real: ~200–400 MB per session
  at rest, more under load. Hosting 100 sessions ≈ 30+ GB RAM. This destroys
  the unit economics if you're not careful.
- **WhatsApp Web required.** If WhatsApp removes Web support (they've
  signalled they want a mobile-first future), the whole approach dies.
- **Multi-device mode (MD).** Since 2021 WhatsApp moved to MD; OpenWA supports
  it but historically the transition was rough.
- **Phone tethering historically required.** With MD this is no longer true —
  the phone can be offline.
- **Breaking changes.** Roughly every 2–8 weeks WhatsApp ships something that
  needs a patch. If the upstream maintainer is slow, your fleet breaks.
- **Linux deployment quirks.** Chromium needs system fonts, libnss, etc.
  Docker images must include them; `puppeteer` and `playwright` Docker
  variants help.

## License — critical and often missed

This is the gotcha:

- Versions before ~v1.10 were MIT.
- The maintainer then moved to a **license-key model** (you needed a paid
  key to unlock certain features, and some builds were encrypted / capability-
  limited without one).
- Subsequent versions have shifted between models.

**Check the LICENSE file of the specific version before depending on it
commercially.** Don't assume the GitHub README represents current licensing.

If we want a permissive license guaranteed, **Baileys (MIT)** or
**whatsapp-web.js (Apache-2.0)** are safer foundations than OpenWA.

## Maintenance status (as of research date)

- Upstream `open-wa/wa-automate-nodejs`: still active but cadence has slowed.
- Several forks exist because of the license drama. Quality varies.
- Community has largely shifted to Baileys for new projects, because Baileys
  doesn't use a browser at all — it talks WhatsApp's WebSocket protocol
  directly.

## How sessions persist

OpenWA uses Puppeteer's `userDataDir` — a directory containing the Chromium
profile, including:

- IndexedDB (where WhatsApp Web stores the auth keys)
- localStorage
- Cookies

To restore a session you copy this directory back into place before launching.
For multi-tenant you need one directory per phone number, and you need them
on durable storage if your containers are ephemeral (S3, EFS, persistent
volumes).

This is significantly more storage and operational complexity than Baileys,
which serialises auth state to a small JSON blob (~5–20 KB).

## How to peek / evaluate it yourself

If we want to validate the library hands-on before committing, a one-hour
spike looks like:

```bash
# in a scratch directory, NOT in this repo
npm init -y
npm install @open-wa/wa-automate
```

```ts
// scratch/test-openwa.ts
import { create, Client } from "@open-wa/wa-automate";

create({ sessionId: "scratch" }).then((client: Client) => {
  client.onMessage(async msg => {
    if (msg.body === "ping") await client.sendText(msg.from, "pong");
  });
});
```

Run it, scan the QR with a burner WhatsApp number, text "ping" to yourself.
What you learn from doing this:
- How long the QR flow takes
- What the session storage looks like
- Real memory footprint
- Whether your environment's Chromium runs cleanly

**Caveats for the spike:**
- Use a number we're willing to lose
- Do not bulk-send anything
- Don't deploy this to a public IP

## When OpenWA is the right choice

- You need access to features only exposed via the WhatsApp Web UI (some
  business catalog operations, some group admin flows historically)
- You already have Puppeteer expertise in-house
- You're OK with the resource and licensing trade-offs

## When it isn't

- High-density multi-tenant (1000+ accounts on one host) — Baileys wins
- Permissive license required from day one — pick Baileys or whatsapp-web.js
- Resource-constrained hosts — Baileys wins by an order of magnitude

## Verdict

OpenWA is well-engineered and was the standard for years, but for a 2026
project the **Baileys vs OpenWA decision should default to Baileys** unless
we identify a specific OpenWA-only feature we need. Detailed comparison in
`03-library-comparison.md`.
