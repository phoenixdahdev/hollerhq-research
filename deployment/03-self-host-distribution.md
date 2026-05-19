# Self-Host Distribution for OSS Users

How OSS users actually run this thing. Critical for adoption.

## The 60-second test

A developer should be able to:

```bash
git clone our-repo
cd our-repo
cp .env.example .env
docker compose up
```

And have a fully working messaging platform on `localhost` in under
5 minutes.

If we don't ship this, we lose the Show HN crowd. They don't read
"production deployment guide" — they run `docker compose up` and bail
if it doesn't work.

## What `docker compose up` runs

```yaml
services:
  postgres:        # primary database
  redis:           # cache + queues
  minio:           # S3-compatible object storage for media
  trigger:         # Trigger.dev self-hosted (3 sub-services)
  api:             # our HTTP API
  worker:          # our session worker (holds WA sessions)
  dashboard:       # our Next.js UI
```

Reasonable defaults baked in:

- Generated random secrets on first run (script that creates `.env` if
  missing)
- Auto-create schema (run migrations on boot)
- Pre-created demo workspace + API key (printed on first start)
- Sample data so the dashboard isn't blank

## What we document

1. **Quickstart** — `docker compose up` happy path
2. **Configuration** — environment variables reference
3. **Production deployment** — running this on Hetzner, on Fly.io, on
   Kubernetes
4. **Upgrades** — pinning versions, migration strategy
5. **Backups** — what to back up (Postgres dumps, MinIO content)
6. **Scaling beyond one VPS** — when and how to split processes across
   machines

## What we do NOT ship

- A managed-cloud-only setup
- A "you need 5 things from AWS" guide
- A Helm chart as the ONLY deployment artifact

These exclude the "try it on my laptop / VPS" crowd which is our
adoption funnel.

## Production hardening

Document for users who go beyond the dev compose:

- TLS with Caddy / Traefik in front
- Postgres backups (`pg_basebackup`, S3 dumps)
- Redis AOF persistence enabled
- MinIO replication for media (or swap to S3/R2)
- Process supervision (systemd / Docker restart policies)
- Resource limits per container
- Monitoring (Prometheus exporters)

## Hosted offering (later)

If/when we build a SaaS, position it as **"Same software, you don't run
it. We do."** Same Docker images we publish OSS, hosted by us. This is
the open-core play. Avoid the anti-pattern of differentiating on
features (cloud-only features); differentiate on operating burden.

## Distribution artifacts we publish

- Docker images on GitHub Container Registry
  (`ghcr.io/your-org/api`, `worker`, `dashboard`)
- `docker-compose.yml` in the repo (dev defaults)
- `docker-compose.prod.yml` for production tweaks
- Helm chart (Phase 7+, if K8s users ask)
- Detailed docs site

## Anti-pattern: requiring our cloud

Some OSS projects gate features behind their cloud (Inngest comes to
mind). Resist this. Same features in OSS as in cloud. Differentiate on
ops, not features.

## Recommended VPS profiles for OSS users

For a single-VPS deployment running the full compose:

| Scale | RAM | CPU | VPS option | Cost |
|---|---|---|---|---|
| Hobby (1–10 WA sessions) | 4 GB | 2 cores | Hetzner CX22 | €4/mo |
| SMB (10–50 WA sessions) | 8 GB | 4 cores | Hetzner CX32 | €8/mo |
| Small business (50–200 sessions) | 16 GB | 8 cores | Hetzner CX42 | €18/mo |
| Mid scale (200+ sessions) | Multi-machine | — | Multiple Hetzner + managed PG | varies |

Hetzner numbers are illustrative; equivalents exist at Linode, DO,
OVH, etc.

## What to ship in v0.1

- `docker-compose.yml` that works on `docker compose up` with zero
  config
- `.env.example` with safe defaults
- README quickstart section
- One blog post: "Self-host {project} in 5 minutes"

Production-deployment guides and Helm charts come after v0.1 traction.
