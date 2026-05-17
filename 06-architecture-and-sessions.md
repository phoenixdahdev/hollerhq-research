# Architecture & Session Management (SMS + WhatsApp)

The hardest engineering problems in this project differ by channel.

- **WhatsApp** is session-heavy: long-lived stateful sessions per number,
  each with auth keys, an open WebSocket or browser, and the ability to
  be invalidated.
- **SMS** is the opposite: stateless HTTP calls to a provider with API
  keys. No sessions, no reconnects, no QR codes.

The architecture has to handle both gracefully, with one API on top.

## The session-as-process model (WhatsApp side)

A WhatsApp session is a long-lived stateful thing:

- Has auth credentials (keys) that must be persisted
- Has an open WebSocket (Baileys) or browser (Puppeteer)
- Receives push events at any moment
- Can become invalidated (logged out from phone, banned)

You can't treat sending a WhatsApp message like an HTTP POST. There's
always a session behind the call, and the session is what really has
the relationship with WhatsApp.

## The provider-call model (SMS side)

SMS is fundamentally different:

- Stateless HTTP call to provider API
- Auth is API key, not paired device
- Provider holds the relationship with the carrier; we just submit
  messages
- Inbound arrives via webhook from provider
- No reconnect logic, no QR codes, no session lifecycle

This means a workload of 100k SMS sends / month from one workspace is
just "call provider 100k times," vs 100 WhatsApp sessions across many
workspaces which is "host 100 long-lived stateful connections."

Different scaling math. Different failure modes. Different code paths.

## Combined topology

```
       ┌─ HTTP API (Hono on Node)
       │       │
       │       ▼ enqueue
       │   Outbound queue (BullMQ on Redis)
       │       │
       │       ▼
       │   Worker pool
       │       │
       │       ├── SMS provider HTTP call (stateless, easy)
       │       │
       │       └── WhatsApp session (stateful, in-memory map)
       │                       │
       │                       ▼
       │              Auto-reconnect / pair / etc.
       │
       └─ Webhook receiver (provider events, WA messages)
                  → inbound queue
                  → customer webhooks
```

API layer is the same. Worker layer holds:

- A simple HTTP client per SMS provider (stateless)
- A `WhatsAppSession` per workspace × phone (stateful)

Worker scaling is dominated by the WhatsApp side. SMS sends are cheap to
parallelize across any worker.

## Topology options (WhatsApp scaling)

### Option 1: Single process, many sessions

A long-running Node process holds N session objects in memory.

**Pros:** simplest; low overhead per session; easy intra-session ops.
**Cons:** process restart = all sessions reconnect; one bad session can
crash everything; doesn't scale horizontally without sharding.
**When:** up to a few hundred sessions of moderate activity.

### Option 2: Worker pool, sessions sharded

Multiple worker processes, each holding a slice of sessions. Router
directs API calls to the right worker.

**Pros:** scales horizontally; isolated failure domains.
**Cons:** need sticky routing; more moving parts.
**When:** outgrowing Option 1.

### Option 3: One container per session

Kubernetes pod / Docker container per WhatsApp number.

**Pros:** total isolation; easy per-tenant billing.
**Cons:** heavy overhead per session; complex orchestration.
**When:** enterprise / single-tenant per customer.

### Recommendation

Start with Option 1 in dev and small-scale production. Design the
session boundary so moving to Option 2 later is a sharding change, not a
redesign. SMS workers can be stateless and elastic from Day 1 because
SMS doesn't need stickiness.

## Session storage (WhatsApp)

### For Baileys

Baileys exposes `useMultiFileAuthState(folder)` (file-backed); you can
write custom auth-state adapters for Redis / Postgres / S3.

Recommended for multi-tenant: a custom adapter that:

- Stores auth state as encrypted JSON per session
- Lives in Postgres `whatsapp_sessions` table
- Encrypted with a per-tenant key derived from a KMS root

```ts
async function makeDbAuthState(sessionId: string): Promise<{
  state: AuthenticationState;
  saveCreds: () => Promise<void>;
}> {
  // load creds + keys from DB, decrypt
  // return shape Baileys expects, with saveCreds that re-encrypts + writes
}
```

Portable across worker nodes (scaled-out architecture) and properly
secured.

### For OpenWA / whatsapp-web.js

You need the entire Chromium `userDataDir` (~50 MB compressed). Options:

- Persistent volume per session (works in Kubernetes)
- Tar + S3 on disconnect, restore on connect (slower, works in
  stateless containers)
- Network filesystem (EFS) — simple but slow first-launch

Materially more complex than Baileys' JSON blob.

## Provider configuration (SMS)

For SMS, "session storage" is really "provider configuration":

```sql
CREATE TABLE sms_providers (
  id UUID PRIMARY KEY,
  workspace_id UUID NOT NULL REFERENCES workspaces(id),
  provider TEXT NOT NULL,             -- 'twilio', 'telnyx', 'plivo'
  encrypted_credentials BYTEA,        -- API key, account SID, etc.
  default_from TEXT,                  -- E.164 sender number
  is_active BOOLEAN NOT NULL DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

Much simpler than WhatsApp session storage. No durable state per send
beyond the message row itself.

## Driver / provider interface (unified)

The abstraction that lets the rest of the system be channel-agnostic:

```ts
export interface MessageDriver {
  channel: 'sms' | 'whatsapp';
  capabilities: {
    media: boolean;
    buttons: boolean;
    templates: boolean;
    groups: boolean;
    pricePerMessageUsd?: number;
  };

  send(req: SendRequest): Promise<SendResult>;

  // Optional — only WhatsApp drivers implement these
  startSession?(workspaceId: string): Promise<SessionStartResult>;
  closeSession?(workspaceId: string): Promise<void>;

  // Event emitter (mostly for inbound)
  on(event: 'message', cb: (e: IncomingMessage) => void): void;
  on(event: 'status', cb: (e: StatusEvent) => void): void;
}
```

Concrete implementations:

- `TwilioSmsDriver` — wraps Twilio SMS
- `TelnyxSmsDriver` — wraps Telnyx SMS
- `BaileysWhatsAppDriver` — wraps Baileys
- `CloudApiWhatsAppDriver` — wraps WhatsApp Cloud API (Phase 5)

The smart router (channel selection) lives one layer above this
interface, picking which driver to call.

## Queueing outbound messages

You should not let API callers send messages synchronously through any
driver. Queue between them:

```
API handler
    │
    ▼
Enqueue message (Redis / BullMQ)
    │
    ▼
Worker dequeues for the right driver / session
    │
    ▼
Per-driver / per-session rate limiter
    │
    ▼
driver.send(...)
    │
    ▼
Status event → update DB → fire webhook to customer
```

Per-channel queue concerns:

- **SMS**: queue for backpressure, idempotency, retry on transient
  provider errors. No per-recipient rate limit needed below provider's
  own throttling.
- **WhatsApp**: queue MUST enforce per-session rate limits (≤1 msg/sec
  default). Bursts trigger bans. Per-session concurrency = 1.

Suggested topology: separate queue namespaces per driver:

- `messaging:sms:outbound`
- `messaging:wa:outbound:{sessionId}` (per-session, rate-limited)

## QR provisioning flow (WhatsApp only)

The weirdest UX in the product:

```
1. User clicks "Connect WhatsApp"
2. API creates a pending session, opens WS, gets QR data
3. API returns QR (or short-lived URL to a QR endpoint)
4. Frontend renders QR; polls or subscribes (SSE / WebSocket) for state
5. User scans → session moves to AUTHENTICATED
6. Auth state persisted, session is now live
```

Hidden complexity:

- QR expires every ~20 seconds; library emits new ones
- If user takes too long, session needs to be re-initialised
- Pairing-code flow exists (Baileys, Aug 2023+): user types a phone
  number, gets an 8-digit code to enter on phone. Often nicer than QR.

Build both. Pairing-code-first as default, QR as fallback.

SMS needs none of this — give us your provider API key, done.

## Inbound message handling (cross-channel)

Different webhook shapes per source:

- Twilio → Twilio's webhook format
- Telnyx → Telnyx's webhook format
- Baileys → in-process event handler
- Cloud API → Meta's webhook format

Each driver normalises to our internal `IncomingMessage` shape:

```ts
type IncomingMessage = {
  channel: 'sms' | 'whatsapp';
  from: string;            // E.164
  to: string;              // E.164
  body: {
    type: 'text' | 'media' | 'reaction' | 'location' | 'contact';
    text?: string;
    media?: { type: string; url: string };
  };
  external_id: string;     // provider's message ID
  received_at: Date;
  raw: unknown;            // original payload, opaque
};
```

Then the same downstream pipeline applies:

- persist to `messages` table
- thread into `conversations` table
- fire customer webhook (unified shape — see `05-multi-channel-strategy.md`)

Webhooks to customers need:

- HMAC signing (so customer can verify origin)
- Retry with backoff (3 tries: immediate, 30 s, 5 m)
- Dead-letter queue + retrieval UI
- Idempotency key so customer can dedupe

## Health, reconnect, ban detection

**WhatsApp side:**

A session can fail in many ways:

- WebSocket disconnects (transient) → auto-reconnect
- Auth invalidated (phone logged out the linked device) → notify user,
  mark session as dead, require re-pair
- Rate-limited / soft-banned → throttle and back off
- Hard-banned → notify user, session is dead, no recovery

Health check loop per session:

- Every 60 s: check liveness of socket
- Track time-since-last-event; if > N min, send a ping
- On disconnect, attempt reconnect with exponential backoff capped at 5 min
- After K failed reconnects, mark as DEGRADED and alert

**SMS side:**

Track provider error rates (4xx, 5xx). Alert if rate spikes — could
indicate misconfiguration, suspended account, or carrier issue. No
"reconnect" concept; nothing to reconnect to.

## Multi-tenant data model (updated for multi-channel)

```sql
-- Workspaces / orgs
CREATE TABLE workspaces (
  id UUID PRIMARY KEY,
  name TEXT NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- API keys per workspace
CREATE TABLE api_keys (
  id UUID PRIMARY KEY,
  workspace_id UUID NOT NULL REFERENCES workspaces(id),
  key_hash TEXT NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- SMS provider configs (one workspace can have multiple, eg. Twilio + Telnyx)
CREATE TABLE sms_providers (
  id UUID PRIMARY KEY,
  workspace_id UUID NOT NULL REFERENCES workspaces(id),
  provider TEXT NOT NULL,             -- 'twilio', 'telnyx'
  encrypted_credentials BYTEA,
  default_from TEXT,
  is_active BOOLEAN NOT NULL DEFAULT true
);

-- WhatsApp sessions (one per phone number per workspace)
CREATE TABLE whatsapp_sessions (
  id UUID PRIMARY KEY,
  workspace_id UUID NOT NULL REFERENCES workspaces(id),
  phone_number TEXT,                  -- E.164, null until paired
  status TEXT NOT NULL,               -- PENDING, AUTHENTICATED, DISCONNECTED, BANNED
  driver TEXT NOT NULL,               -- 'baileys', 'cloud_api'
  encrypted_auth BYTEA,
  last_connected_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Conversations (cross-channel thread, keyed by recipient phone)
CREATE TABLE conversations (
  id UUID PRIMARY KEY,
  workspace_id UUID NOT NULL REFERENCES workspaces(id),
  recipient_phone TEXT NOT NULL,      -- E.164 — same identifier across channels
  last_message_at TIMESTAMPTZ,
  metadata JSONB,
  UNIQUE (workspace_id, recipient_phone)
);

-- Messages (both channels, both directions)
CREATE TABLE messages (
  id UUID PRIMARY KEY,
  workspace_id UUID NOT NULL REFERENCES workspaces(id),
  conversation_id UUID NOT NULL REFERENCES conversations(id),
  channel TEXT NOT NULL,              -- 'sms' or 'whatsapp'
  direction TEXT NOT NULL,            -- 'outbound' or 'inbound'
  driver TEXT NOT NULL,               -- 'twilio', 'telnyx', 'baileys', 'cloud_api'
  to_number TEXT,
  from_number TEXT,
  body JSONB NOT NULL,
  status TEXT NOT NULL,               -- QUEUED, SENT, DELIVERED, READ, FAILED
  external_id TEXT,
  routing_reason TEXT,                -- 'static:sms', 'fallback:sms_after_wa_fail', 'smart:wa_in_window'
  cost_usd_cents INT,
  error JSONB,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Webhook endpoints
CREATE TABLE webhooks (
  id UUID PRIMARY KEY,
  workspace_id UUID NOT NULL REFERENCES workspaces(id),
  url TEXT NOT NULL,
  secret TEXT NOT NULL,
  events TEXT[] NOT NULL,
  channel_filter TEXT[]               -- null = all channels; ['sms'] = SMS only
);

-- Opt-out / blocklist (TCPA compliance, STOP handling)
CREATE TABLE opt_outs (
  id UUID PRIMARY KEY,
  workspace_id UUID NOT NULL REFERENCES workspaces(id),
  phone_number TEXT NOT NULL,         -- E.164
  channel TEXT NOT NULL,              -- 'sms' or 'whatsapp' or 'all'
  reason TEXT,                        -- 'user_stop', 'manual', 'bounce'
  created_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE (workspace_id, phone_number, channel)
);
```

Single `messages` table for both channels avoids the trap of two
parallel pipelines for the same conceptual thing. Channel and driver are
columns, not separate tables.

## Putting it together

```
┌────────────────────────────────────────────────────────────────────┐
│  Customer's app                                                      │
│      │                                                               │
│      ▼ HTTPS                                                         │
│  ┌─────────────────────┐                                             │
│  │  API (Hono on Node) │  ─────► Postgres (state, conversations)    │
│  │  (apps/api)         │  ─────► Redis (queues, sessions)           │
│  └─────────────────────┘                                             │
│             │                                                        │
│             ▼ Channel router decides + enqueues                      │
│  ┌─────────────────────┐                                             │
│  │  Workers (apps/worker)│                                            │
│  │  ├─ SMS driver pool │  ─────► Twilio / Telnyx API                │
│  │  └─ WA session pool │  ◄────  WhatsApp servers (WebSocket)       │
│  │                     │  ─────► Customer webhooks (both channels)  │
│  └─────────────────────┘                                             │
└────────────────────────────────────────────────────────────────────┘
```

Phase 0 doesn't need separate API and worker — they can be one process.
But the separation should be visible in code from day one so the split
is mechanical when needed.

## Open architectural questions

1. **One worker process per channel, or mixed?** Mixed at small scale.
   Split at larger scale — WhatsApp sessions benefit from stickiness,
   SMS is stateless and can spread.
2. **Where does the smart router live?** Inside the API process
   (synchronous decision before enqueue) or inside workers (decided when
   dequeued)? Lean: in API. Cheaper to keep policy decisions out of
   workers.
3. **Conversation locking**: if two messages enqueue to the same
   recipient from different processes, do we serialize? Yes
   per-recipient — avoids out-of-order delivery weirdness.
4. **Local development**: how do we test without burning real SMS / WA
   credits? Need `MockSmsDriver` and `MockWhatsAppDriver` that record
   and replay. Ship as `@your-org/messaging-mocks`.
