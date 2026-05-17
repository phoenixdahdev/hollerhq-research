# Multi-Channel Strategy

This is where the product idea actually lives.

Wrapping SMS or WhatsApp alone is a commodity. Twilio does both. Most
BSPs do both. The novel thing isn't the channels — it's making them feel
like **one messaging primitive** with smart behaviour underneath.

## The one-sentence pitch

> Send a message to any phone number; the SDK picks WhatsApp if
> available, falls back to SMS if not, threads them as one conversation,
> and gives the developer one API and one webhook shape.

That's the wedge. Nobody ships this well today.

## Why a unified API is the actual product

The honest answer to "why bother building this?":

- **Cost optimization.** WhatsApp service conversations are free; SMS
  is $0.004–$0.05. Letting the platform pick WhatsApp where possible
  saves customers money — sometimes a lot of money.
- **Higher delivery rates.** WhatsApp doesn't get spam-filtered; SMS
  does. If WhatsApp delivers, prefer it.
- **Richer messages where possible.** Media, buttons, templates —
  degrade to plain text on SMS.
- **Lower cognitive load.** One API, one webhook, one billing surface,
  one inbox.

If a developer has to pick channel themselves every time they call
`send()`, we haven't actually helped them. They could've just used Twilio.

## What "smart" routing actually means

Three increasingly sophisticated routing strategies, all options the
developer should be able to pick from:

### Strategy 1: Static (caller-decides)

```ts
await messaging.send({
  to: '+15558675309',
  channel: 'whatsapp',          // or 'sms'
  body: 'Your order shipped',
});
```

Caller picks. We just route. Useful for compliance-heavy senders who
know exactly which channel a message needs (e.g., OTP must go via SMS).

### Strategy 2: Prefer-with-fallback

```ts
await messaging.send({
  to: '+15558675309',
  channels: ['whatsapp', 'sms'],   // try WA first, fall back to SMS
  body: 'Your order shipped',
});
```

We try WhatsApp first; if it fails (not on WhatsApp, ban, error), we
send via SMS. Acknowledges that delivery matters more than channel.

### Strategy 3: Smart (we decide)

```ts
await messaging.send({
  to: '+15558675309',
  body: 'Your order shipped',
});
```

We pick based on:

- Has the recipient previously responded on WhatsApp? Use WA.
- Is the recipient's number known to be on WhatsApp (via a check)?
  Use WA.
- Is the cost differential favorable? Use cheaper.
- Is this user in a 24-hour WhatsApp window? Use WA (it's free in that
  window).
- Else: SMS.

This requires us to track conversation history per recipient. It's a real
investment but it's also what makes the product magical.

## The "is this number on WhatsApp?" check

WhatsApp Cloud API does NOT expose a "does this number have WhatsApp?"
check. Baileys / unofficial libraries DO — `sock.onWhatsApp([phone])`
returns whether the number is registered.

This is a small but real advantage of the unofficial path: existence
checks without sending. Caveat: doing many of these checks fast is itself
a ban signal. Cache aggressively, rate-limit, batch.

Useful pattern: lazy check. First time we send to a number, try WhatsApp
optimistically. Record the result. Subsequent sends use the cached answer.

## Unified webhook event shape

The other half of "feels like one channel" is incoming.

```ts
// What every inbound webhook event looks like
{
  type: 'message.received',
  channel: 'whatsapp' | 'sms',
  conversation_id: 'conv_...',     // maintained across channels for the same recipient
  from: '+15558675309',
  to: '+15551234567',
  body: { type: 'text', text: 'thanks!' },
  external_id: 'wamid....' | 'SM....',
  timestamp: '2026-05-17T...',
  raw: { /* original payload, untranslated */ }
}
```

The shape is the same regardless of channel. Developers register one
webhook handler and don't care whether the reply came via SMS or
WhatsApp — they get the same event, and the `conversation_id` ties it
to the outbound message.

## Conversation continuity

The hardest unified-API problem: a user got an SMS, replied via
WhatsApp. Or vice versa. How do we present that as one conversation?

Strategies:

- **Phone-number-keyed conversations.** All messages to/from this
  number = one conversation. Works because phone numbers identify both
  channels.
- **Time-windowed sessions.** Conversation auto-closes after N hours of
  silence; new burst = new conversation. Matches how humans actually
  use messaging.
- **Explicit conversation IDs.** Caller passes a conversation ID with
  each send; we surface it on inbound.

**Lean:** phone-number-keyed by default + optional explicit IDs for
callers who care (e.g., CRM integrations).

## Unified rich-message shape

WhatsApp has templates, buttons, lists, media. SMS has text only. How
do we express a "rich message" in our SDK?

```ts
await messaging.send({
  to: '+15558675309',
  body: {
    text: 'Your order shipped! Tracking: ABC123',
    media: { type: 'image', url: 'https://...' },         // WA only
    buttons: [{ label: 'Track', url: '...' }],            // WA only
    fallback: 'Your order shipped: ABC123 https://...',   // used when channel can't render rich
  }
});
```

Degradation rules:

- If channel = SMS → send `fallback` (or strip rich → text)
- If channel = WhatsApp + has matching template → use template
- If channel = WhatsApp + no template → send rich inline

The fallback design lets developers express rich-when-possible without
two API calls.

## Cost and billing surface

Even at the SDK layer, developers want to know what a send will cost.

```ts
const estimate = await messaging.estimate({
  to: '+15558675309',
  body: '...',
  channels: ['whatsapp', 'sms'],
});
// → {
//     likely_channel: 'whatsapp',
//     estimated_cost_usd: 0.00,
//     sms_fallback_cost: 0.0079
//   }
```

Implementation: provider rate tables + 24-hour-window detection for WA.
Doesn't need to be precise; needs to be directional.

## API surface sketch (rough)

```ts
import { Messaging } from '@your-org/sdk-node';

const messaging = new Messaging({
  providers: {
    sms: { provider: 'telnyx', apiKey: process.env.TELNYX_API_KEY },
    whatsapp: { provider: 'baileys', sessionId: 'workspace_42' },
  },
});

// Smart routing (we decide)
await messaging.send({ to: '+15558675309', body: 'Hi' });

// Explicit channel
await messaging.sms.send({ to: '+15558675309', body: 'Hi' });
await messaging.whatsapp.send({ to: '+15558675309', body: 'Hi' });

// One handler for both channels
messaging.on('message', (event) => {
  console.log(event.channel, event.from, event.body);
});

// Conversation lookup (cross-channel)
const conv = await messaging.conversations.get({ phone: '+15558675309' });
for (const m of conv.messages) {
  console.log(m.channel, m.direction, m.body);
}
```

Day 1 doesn't need all of this — `send()` and event handlers come
first; conversations and smart routing come once the basics work.

## How current players do (and don't) do this

### Twilio Conversations API

Twilio's attempt. It works but the DX is rough:

- Separate product, separate billing surface
- API is REST-y with much SDK ceremony
- Configuration is heavy (services, participants, channels)
- Smart routing isn't really smart — caller still picks the channel
- Auto-generated SDK; doesn't feel hand-crafted

Twilio Conversations *exists* but it's clearly the product nobody loves
internally. Room to do this better.

### MessageBird / Bird

Bird (rebranded MessageBird) has a multi-channel SDK. Better positioned
than Twilio Conversations, but:

- Enterprise sales motion
- Heavy onboarding
- No OSS / self-host story
- Pricing is opaque

### Sinch, Infobip

Both ship multi-channel for enterprise. Same patterns — sales-led,
heavy, no DX love.

### Open source

- **Chatwoot** — multi-channel inbox UI, but inbox-focused (agents
  working conversations), not SDK-focused (developers sending messages
  programmatically).
- **Novu, Knock, Courier** — notification infrastructure, multi-channel
  (SMS, email, push, in-app, sometimes chat). Closest DX comparison. But
  they're one-way (notifications, not conversations). Sending is great;
  receiving / inbound webhooks are weak or absent.
- **No one** ships a great OSS conversational multi-channel SDK with
  SMS + WhatsApp + smart routing. This is our hole.

## Recommended product shape

- **One SDK surface**: `@your-org/messaging` exposes `messaging.send`,
  `messaging.sms.*`, `messaging.whatsapp.*`, and shared event handling.
- **Two channel families to start**: SMS (one or two providers) +
  WhatsApp (one driver to start, Baileys).
- **Three routing strategies, in order of difficulty**: static, fallback,
  smart.
- **One conversation model** across both channels, keyed by recipient
  phone.
- **One webhook event shape** unified across channels.

Build static + fallback first. Smart routing is a v0.3 thing once we
have conversation history to base it on.

## Open product questions

1. **Do we ship our own number provisioning, or pass-through provider
   numbers?** Pass-through is simpler (use customer's existing Twilio
   / Telnyx numbers). Own provisioning is more product-y but means
   dealing with 10DLC ourselves.
2. **Should `messaging.send()` block until delivered, or return on
   queued?** Return on queued (with optional `await delivered`
   helper). Blocking is bad UX for HTTP handlers.
3. **How explicit is the "we picked this channel" feedback?**
   Always include the chosen channel in the response, plus the
   reason (e.g., `"whatsapp:in_window"`). Helps debugging and trust.
4. **What about email as a third fallback?** Tempting but: different
   identifiers (email vs phone), different consent regimes, different
   webhook shapes. Treat email as a Phase 4+ extension, not a fallback
   layer.
