# Relational Database

## Postgres — the choice

Postgres is the default. No reason to consider alternatives:

- ACID for billing / counters
- JSONB for flexible message payloads
- Full-text search for inbox features later
- Partitioning for the messages table at scale
- Standard, well-known, broadly supported

Don't consider MySQL, SQLite-in-production, or NoSQL stores at this
layer.

## Postgres host options

| Option | Pros | Cons |
|---|---|---|
| **Self-hosted (Docker)** | Free, total control | We manage backups, upgrades, replicas |
| **Neon** | Serverless, scale-to-zero, branching for dev | Newer; cold starts on idle (mitigated with provisioned compute) |
| **Supabase** | Postgres + extras (auth, realtime, storage), open source | We don't need most of the extras; coupling |
| **Vercel Postgres** | Vercel-integrated | Vercel-coupled, not great for self-host story |
| **AWS RDS / Aurora** | Enterprise-grade | Expensive, AWS-coupled |
| **DigitalOcean / Render managed PG** | Cheap, simple | Less featureful than Neon |

**Lean for OSS users:** docs default to Docker compose Postgres
self-hosted. We provide everything via a standard Postgres connection
URL.

**Lean for our own SaaS / dev:** Neon. Branching for staging/dev, low
ops, generous free tier.

## ORM choice — the big TS-stack decision

Three real options in the modern TS Postgres space.

### Drizzle

[orm.drizzle.team](https://orm.drizzle.team).

**Strengths:**

- Lightweight, SQL-shaped TS API
- Generates migrations from schema
- No code generation step (Prisma's pain point)
- Edge runtime compatible
- TS types are excellent
- Growing fast in 2024–2026

**Weaknesses:**

- Younger; smaller community than Prisma
- Relations API was rough early; better now
- Migration tooling improving but not bulletproof

### Prisma

[prisma.io](https://www.prisma.io).

**Strengths:**

- Mature, polished, the "default" TS ORM
- Excellent docs
- Generated client is ergonomic

**Weaknesses:**

- Code generation step in CI is painful
- Generated client is heavy
- Edge runtime support added but constrained
- Performance has historical complaints (improved with engine
  rewrites)
- "Magic" can be a footgun for complex queries

### Kysely

[kysely.dev](https://kysely.dev).

**Strengths:**

- Pure type-safe SQL builder
- No code generation
- You write SQL-shaped JS; compiler validates types
- Lightweight

**Weaknesses:**

- More verbose than ORM-style libraries
- No migrations tooling (use e.g. node-pg-migrate)
- Smaller community

## Recommendation

**Drizzle.** Best balance of TS-native ergonomics, no codegen, edge
support, growing community. Aligns with our "TS-first, modern stack"
positioning.

Prisma if we want maximum maturity at the cost of codegen pain.
Kysely if we want maximum SQL purity and don't mind being more verbose.

## Schema reference

See `research/06-architecture-and-sessions.md` for the rough schema
sketch. Drizzle schema definitions for the same shapes would live in
`packages/db/schema/`.

## Migrations

- Use Drizzle Kit for schema-driven migrations
- Migrations are checked into git
- Production deploys run migrations before app start
- For destructive migrations, manual review (not in any CI)

## Partitioning (future)

The `messages` table will grow fast. Plan from Day 1 to partition by
`created_at` (monthly partitions). Drizzle doesn't fully support
declarative partitioning but you can use raw SQL migrations.

Not a Day 1 concern; design for it.
