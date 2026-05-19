# Hosting Options

For our own deployment (SaaS / demo). OSS users get separate guidance
in `03-self-host-distribution.md`.

## Tier 1: container-as-a-service platforms

### Fly.io

[fly.io](https://fly.io).

**Strengths:**

- Runs Docker containers globally (edge presence in many regions)
- Persistent volumes
- WS supported
- Internal networking between apps in same org (private 6PN /
  Flycast)
- Reasonable pricing
- Good for stateful workloads
- Good for "I want a small footprint that runs reliably"

**Weaknesses:**

- Has had operational reliability incidents (improving)
- Postgres offering is OSS but you manage
- Smaller team than AWS-scale providers

**Fit:** very good. Lean choice for our own production.

### Railway

[railway.app](https://railway.app).

**Strengths:**

- Excellent DX; one of the easiest container hosts
- Managed PG / Redis built-in
- Per-environment branching
- GitHub-deploy

**Weaknesses:**

- Pricier than Fly for sustained workloads
- Less control over regions / networking

**Fit:** great for demo/dev environments; pricier for sustained
production.

### Render

[render.com](https://render.com).

**Strengths:**

- Simple, polished DX
- Managed PG, Redis included
- Background workers as first-class
- Cron jobs built-in (we don't need this, Trigger.dev handles)

**Weaknesses:**

- Mid-pricing
- Limited regions

**Fit:** good alternative to Railway.

### DigitalOcean App Platform

**Strengths:** mature DO ecosystem, managed PG nearby
**Weaknesses:** less differentiated DX
**Fit:** OK fallback.

## Tier 2: Kubernetes

### Self-managed K8s (DOKS, EKS, GKE)

**Strengths:**

- Maximum flexibility
- Industry standard at scale

**Weaknesses:**

- Big operational burden
- Overkill for V1
- Not "casually" hostable

**Fit:** consider only at significant scale (thousands of sessions).

## Tier 3: VPS

### Hetzner / OVH / DigitalOcean Droplet

**Strengths:**

- Cheapest per dollar
- Total control
- Hetzner specifically is famously cheap and reliable

**Weaknesses:**

- We manage everything (TLS, restarts, OS updates, backups)
- No managed PG/Redis on cheapest VPS

**Fit:** great for OSS docs ("you can run this on a $20 Hetzner VPS").

## Recommendations

### For our own dev / staging

Railway. Branch-per-PR, managed services, fast iteration.

### For our demo / Show HN env

Fly.io. Cheap, reliable enough, can run multiple apps in one org.

### For our SaaS production (when we get there)

Fly.io to start, evaluate K8s if we cross ~1k active sessions.

### For OSS users

Docker compose on a single VPS (Hetzner $20 box runs this nicely).
Documented in `03-self-host-distribution.md`.

## Cost rough estimate

For our own SaaS with ~100 active WA sessions, 100k msg/mo:

| Layer | Provider | Monthly |
|---|---|---|
| Compute (3 small VMs) | Fly.io | ~$40 |
| Postgres | Fly Postgres or Neon | ~$25 |
| Redis | Upstash / Redis Cloud | ~$15 |
| Object storage | Cloudflare R2 | ~$10 |
| **Total** | | **~$90/mo** |

At Year-3 mid-market scale that probably becomes $500–$1500/mo. Still
fine for a SaaS business.

## What we explicitly do NOT recommend

- **Vercel** — wrong runtime; can't host the session worker
- **Cloudflare Workers** — same constraint
- **AWS Lambda** — same constraint
- **Heroku** — declining quality, expensive, no compelling reason

These are all great for stateless web apps; ours isn't stateless.

## Multi-region considerations

V1: single region (US-East or EU-West, pick based on first customer
region).

Later: multi-region for the API tier is cheap; multi-region for the
session worker is hard because each WA session is pinned to a specific
phone number's region. Don't overthink this until forced by latency
data.
