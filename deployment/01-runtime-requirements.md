# Runtime Requirements

Concrete needs from the hosting layer.

## Processes

| Process | Long-lived | Stateful | Scaling |
|---|---|---|---|
| API (Hono) | Yes | No | Horizontal, stateless |
| Engine task runners (Trigger.dev workers) | Yes | No | Horizontal |
| Engine supervisor (Trigger.dev) | Yes | Coordinator | Singleton |
| Engine webapp (Trigger.dev dashboard) | Yes | No | Single instance |
| Session worker (holds WA sessions) | Yes | **Yes** (in-memory sessions) | Sticky horizontal sharding |
| Webhook delivery worker | Could be inside engine | No | Horizontal |
| Dashboard (Next.js) | Yes | No | Horizontal |

## Networking

- **API**: external HTTPS, internal to Postgres, Redis, engine, session
  worker
- **Engine**: internal to Postgres, Redis, session worker; outbound to
  customer webhooks
- **Session worker**: internal to Postgres, Redis; **outbound WebSocket
  to WhatsApp servers** (the constraint that kills serverless)
- **Dashboard**: external HTTPS, internal to API

## Persistent state

- Postgres: durable, managed
- Redis: hot, durable enough (we accept some loss tolerance)
- Object storage: for media (see `storage/03-object-storage.md`)
- **No local disk required** if session auth state is in Postgres

## Resource estimates

| Process | Memory baseline | CPU baseline | Notes |
|---|---|---|---|
| API | ~150 MB | Low | Scales with request rate |
| Engine task runner | ~250 MB | Moderate | Per worker process |
| Session worker | ~50 MB base + ~50 MB per Baileys session | Moderate per active session | Real cost driver |
| Postgres | ~512 MB minimum | Low at our scale | Managed handles this |
| Redis | ~256 MB | Low | Managed handles this |
| Dashboard | ~200 MB | Low | Serves UI |

A V1 deployment with ~100 active WA sessions on Baileys = ~10 GB RAM
total across all services. Fits on a $40–80/month VPS or small managed
instance.

## Scaling characteristics

- **API**: linear with request rate, stateless, scale to N
- **Engine workers**: linear with job rate, scale to N
- **Session worker**: linear with active sessions, sharded; sessions
  pinned to specific worker instance (need sticky routing or a
  session-locator registry in Redis)
- **Database**: vertical first, then read replicas
- **Redis**: vertical; sharding is painful, avoid until really
  necessary

## Networking patterns

```
[ internet ]
     │
     ▼
  ┌──────────┐
  │  Caddy / │    TLS termination
  │  Traefik │    Routes to API + dashboard
  └────┬─────┘
       │
       ├───► api (port 8080)
       │
       └───► dashboard (port 3000)

  Internal:
  api ──► engine (port 4321, Trigger.dev internal)
  api ──► worker (port 4000, internal RPC)
  engine ──► worker (same)
  all ──► postgres (5432)
  all ──► redis (6379)
```

Internal services should not be on the public internet. On Fly.io, use
the `flycast` private network. On Railway/Render, services in same
project share private network. On Kubernetes, ClusterIP services.

## Persistent connections

The session worker holds outbound WebSockets to `wss://web.whatsapp.com/`.
Hosting requirements:

- TCP egress allowed
- Idle TCP connections not killed by load balancer (Fly.io: OK; many
  cloud LBs kill idle TCP at 60s — not the issue for us because traffic
  is regular)
- Sufficient file descriptors per process (`ulimit -n` ≥ 65536)
