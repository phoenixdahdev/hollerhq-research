# Risks & Mitigation (SMS + WhatsApp)

Both channels carry distinct risks that need to be priced in and
communicated.

## WhatsApp risks

### 1. WhatsApp Terms of Service violation (unofficial path)

WhatsApp's [Terms](https://www.whatsapp.com/legal/business-terms/)
prohibit "unauthorized" interaction with their service, including:

- Bulk or automated messaging via non-approved means
- Reverse engineering
- Using WhatsApp for spam or unsolicited messaging

Using Baileys, OpenWA, or whatsapp-web.js technically violates this.
Meta has not litigated against the *libraries* (suing OSS authors is bad
PR) but individual users running these tools have been suspended.

**Mitigation:**

- Explicit that users are responsible for their use
- Don't market the product for spam or cold outreach
- Document opt-in practices
- ToS disclaim responsibility for account actions
- Refuse paying customers using it for spam (signalling matters legally)

### 2. WhatsApp account bans

WhatsApp flags accounts that:

- Send many messages quickly
- Send to many new (un-contacted) numbers
- Have high block / report rates
- Show "non-human" patterns (perfectly regular intervals, instant
  replies, etc.)

Bans range from:

- **Soft block** (24-hour silent throttle)
- **Temporary ban** (24h–7d)
- **Hard ban** (number permanently disabled — recovery via Meta
  support, often hopeless)

**Mitigation patterns to bake into the product:**

- Per-session rate limits enforced by the queue (configurable,
  conservative defaults)
- "Warmup" mode for new sessions: low limits for the first 1–2 weeks,
  gradually ramping
- Jitter on send intervals (avoid perfectly-regular cadences)
- Honour user-side blocks: stop sending to numbers that have blocked
  the sender
- Detect ban-like behaviour (deliveries silently failing) and alert
  the user
- Account-health dashboard

**Realistic defaults:**

- New session: 20 messages / day for first 3 days, 100 / day for next
  4, then unlimited within rate cap
- Rate cap: 1 message / sec with 200–600 ms jitter
- Cooldown after N consecutive un-delivered messages

### 3. WhatsApp protocol changes

WhatsApp ships updates regularly. When they make breaking changes,
unofficial libraries need patches.

**Mitigation:**

- Pin library versions; don't auto-upgrade
- Watch upstream (Baileys, OpenWA) GitHub for issues
- Runbook for "library is broken, what now"
- Canary session that pings periodically — if it fails, alert
- Weekly regression test against a real burner number

## SMS risks (different beast, different bucket)

### 1. TCPA (US) — $500–$1,500 per unconsented commercial message

The single biggest US-specific risk. Civil liability, class actions are
common, plaintiff bar is well-organised.

**What counts as "consent":**

- **Express written consent** for marketing (very strict; signed
  agreement or equivalent affirmative action)
- **Established business relationship** allows some transactional /
  service messages without explicit opt-in
- **OTP / 2FA messages** generally exempt (they're requested by the
  user initiating login)

**Mitigation we should ship:**

- STOP keyword handling — auto-detect STOP, UNSUBSCRIBE, OPT-OUT and
  add to blocklist
- Document opt-in collection patterns in onboarding
- Blocklist checked on every send (refuse to send to opted-out numbers)
- Audit log of consent capture (for legal defence)
- Default templates that include opt-out language

### 2. A2P 10DLC registration friction (US)

Not a "risk" but a major friction. Without registration:

- T-Mobile blocks unregistered traffic outright
- AT&T and Verizon throttle aggressively
- Carrier fees on every message

**Mitigation:**

- Document the 10DLC registration flow clearly
- Optionally: hosted onboarding wizard that walks customers through
  TCR brand + campaign registration
- Note in docs: toll-free verification as alternative
- For Day 1 demo / OSS use: be clear that hobby / test usage may hit
  unregistered-traffic blocks

### 3. Carrier filtering (silent)

Even with registration, carrier spam filters can silently drop:

- Messages containing certain keywords (financial, gambling, certain
  political terms)
- Messages with URLs (especially shortened URLs)
- Messages from new numbers with thin reputation

You see "delivered" from the provider but the recipient never sees it.

**Mitigation:**

- Surface delivery vs receipt distinction in dashboards
- Document that "delivered to carrier" ≠ "shown to user"
- Encourage long-domain URLs over `bit.ly`-style shorteners
- Encourage warming new numbers slowly

### 4. EU / UK / regional consent regimes

- **GDPR (EU)**: lawful basis required for processing recipient phone
  numbers. Marketing requires explicit consent.
- **PECR (UK)**: marketing SMS requires opt-in or soft opt-in
  (existing customer).
- **CASL (Canada)**: explicit opt-in. Penalties up to $10M per
  violation.
- **LGPD (Brazil)**: similar to GDPR.
- **India DLT**: registration of templates required before sending.
- **Australia Spam Act**: opt-in required.

**Mitigation:**

- Default to opt-in patterns in docs
- Provide regional helpers if we ever go heavy on a region
- DPA template for customers using us as a processor

### 5. Number-level reputation

US carriers track sender reputation. Bad behaviour → throttling or block.

**Mitigation:**

- Warm-up patterns for new numbers (low volume first 2 weeks)
- Monitoring of delivery rates per number
- Alert customers when their number's reputation degrades

## Cross-channel: legal exposure (jurisdiction-specific)

- **US**: WhatsApp's parent has sued spammers (NSO Group, others) but
  not unofficial library authors. TCPA exposure for SMS spammers is
  civil and substantial.
- **EU**: GDPR concerns for both channels. Become a processor when
  storing message data on behalf of users.
- **India**: TRAI rules for SMS (DLT). Limited WhatsApp commercial
  rules.
- **Brazil**: Active WhatsApp commercial use; LGPD applies to SMS;
  permissive overall.
- **UK**: PECR for SMS; similar for WhatsApp marketing.

**Mitigation:**

- Don't operate as SaaS without legal review in target jurisdictions
- Position OSS-first — software is software; what users do is their
  responsibility
- If we run SaaS, accept only legitimate business use cases and reserve
  right to terminate

## Combined risk dashboard

For both channels, we should provide visibility:

- Delivery rate (per channel, per number)
- Block rate
- Opt-out rate
- Cost per message
- Estimated reputation health (heuristic)

"Safety as a product feature" isn't just rules in code — it's visibility
to the user.

## Risk profile by audience

| Audience | WA risk | SMS risk | Should we serve? |
|---|---|---|---|
| Devs building personal automations | Low | Low (low volume) | Yes |
| SMB customer support / sales | Medium | Medium (TCPA care) | Yes, with warnings |
| Marketing agencies sending campaigns | High | High (TCPA exposure) | No, or Cloud API + verified flows |
| Cold outreach / spam | Very high (bans) | Very high (TCPA) | No, refuse use case |
| OTP / 2FA providers | N/A | Low (regulated, exempt) | Yes (great fit) |
| Transactional alerts (banks, ecommerce) | Low | Low (with consent) | Yes |

OTP and transactional alerts are notably good fits because the
regulatory carve-outs make them straightforward.

## Practical hierarchy

- **Lowest risk**: Cloud API (WhatsApp) + SMS via toll-free or
  10DLC-registered long codes, for transactional / OTP / service use
  cases.
- **Medium risk**: Baileys (WA) + SMS, for legitimate customer support
  and light engagement workflows.
- **High risk**: bulk marketing on either channel. Avoid.

## How others handle it

- **Wassenger**: explicit "unofficial" disclaimer; charges premium
  prices that imply customers pay for operational support.
- **Baileys**: README disclaimer; mitigation is on you.
- **Evolution API**: disclaimer; some rate limiting in code but not
  aggressive.
- **Twilio**: strict policy enforcement; suspends accounts for TCPA-
  violating patterns. Heavy compliance tooling.
- **Telnyx, Plivo**: similar policy posture, less compliance tooling.

The pattern: disclaim clearly, give users tools to behave well, accept
that you can't enforce it.

## Recommendations for our project

1. **Front-page disclaimer covering both channels.** Honest about TCPA
   and WA ToS. Honesty is also good marketing — devs respect clarity.
2. **Safety defaults ON.** Warmup + rate limits (WA), STOP handling +
   blocklist (SMS) — on by default, opt-out to relax.
3. **Use-case allowlist in marketing copy.** "Built for: customer
   support, transactional alerts, OTP, personal assistants, internal
   bots. NOT for: cold outreach, marketing blasts."
4. **Cloud API path as the "responsible WhatsApp default."** When we
   add the Cloud API driver, recommend it in docs even though Baileys
   ships first.
5. **Per-number health dashboard.** WA session status, SMS reputation,
   delivery rates, opt-out rates. Visible safety beats invisible safety.
6. **Compliance helpers as first-class features.** Auto-STOP, consent
   logs, opt-out lists — not add-ons, table-stakes.
