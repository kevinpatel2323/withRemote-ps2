# Data Sync Pipeline

Ingests records from **HubSpot**, **Stripe**, and **Google Calendar** into a single
normalized PostgreSQL schema, with incremental sync, stale-cursor fallback to backfill,
idempotent writes, and per-source fault isolation.

See [`PLAN.md`](./PLAN.md) for the full design and [`EXECUTION.md`](./EXECUTION.md) for the
milestone/parallelization plan.

## Stack

TypeScript · NestJS 11 · PostgreSQL 16 · Drizzle ORM · pg-boss (Postgres-backed queue) ·
`@nestjs/schedule` · Zod · pino. Deploys to Render free tier (single web service + free Postgres).

## Quickstart (local)

```bash
# 1. Install deps
npm install

# 2. Start Postgres — either Docker...
docker compose up -d postgres
#    ...or use a local Postgres and create the role/db:
#    psql postgres -c "CREATE ROLE sync LOGIN PASSWORD 'sync' SUPERUSER;"
#    psql postgres -c "CREATE DATABASE syncdb OWNER sync;"

# 3. Configure env
cp .env.example .env   # defaults already point at the local Postgres above

# 4. Apply migrations
npm run db:migrate

# 5. Run
npm run start:dev      # watch mode
# or: npm run build && npm start

# 6. Verify
curl -s localhost:3000/health   # -> {"status":"ok","db":"up",...}
```

## Scripts

| Script | Purpose |
|---|---|
| `npm run start:dev` | Run with watch + pretty logs |
| `npm run build` | Compile to `dist/` |
| `npm test` | Run unit + integration specs (integration needs Postgres) |
| `npm run lint` | ESLint + Prettier (`--fix`) |
| `npm run db:generate` | Generate a Drizzle migration from `src/db/schema.ts` |
| `npm run db:migrate` | Apply migrations (local, via ts-node) |
| `npm run db:migrate:prod` | Apply migrations from compiled output (Render) |

## HTTP endpoints

| Method | Path | Auth | Purpose |
|---|---|---|---|
| GET | `/health` | — | Liveness + DB check |
| POST | `/webhooks/stripe` | Stripe signature | Stripe events (signature-verified) |
| POST | `/webhooks/google` | channel headers | Google Calendar push → incremental sync |
| POST | `/webhooks/hubspot` | HubSpot signature | HubSpot contact events |
| GET | `/admin/metrics` | `Bearer ADMIN_TOKEN` | Ledger aggregates + reconciliation status |
| POST | `/internal/sync` | `Bearer ADMIN_TOKEN` | Manual sync (all sources, isolated). `?full=true` forces backfill |

Example: `curl -H "Authorization: Bearer $ADMIN_TOKEN" localhost:3000/admin/metrics`

## Layout

```
src/
  config/        env validation (Zod)
  db/            Drizzle schema, connection module, migrate runner
  common/        NormalizedRecord + content-hash
  records/       version-guarded upsert repository (idempotency)
  connectors/    SourceConnector contract + stripe / google / hubspot
  sync/          SyncRunner (fetch→normalize→upsert→ledger), state + run repos
  queue/         pg-boss (Postgres-backed job queue)
  orchestrator/  per-source dispatch + in-process scheduler (fault isolation)
  webhooks/      signature-verified controllers + replay-dedup ledger
  admin/         /admin/metrics + /internal/sync (token-guarded)
  health/        /health endpoint
drizzle/         generated SQL migrations
```

## Design tradeoffs

Every choice below is stated as **decision → why → what it costs**. The deployment constraint
(Render free tier) drives several of them.

- **One `records` table with a `canonical_type` discriminator, not per-entity tables.**
  Satisfies "one normalized schema" and keeps the upsert/idempotency machinery in a single place;
  the long tail of source-specific fields lives in `attributes`/`raw` (JSONB).
  *Cost:* type-specific columns (`amount`, `start_at`, …) are nullable and not enforced per type;
  strongly-typed per-entity tables would catch more at the schema level. (See `PLAN.md` §4.)

- **Postgres *is* the job queue (pg-boss) — no Redis.**
  Render's free Redis evicts keys and has no persistence, which is unsafe for a durable queue;
  pg-boss gives durable jobs, retries, backoff, and a dead-letter queue inside the DB we already
  run. *Cost:* queue throughput is bound to Postgres (fine at this scale, not for high-volume).

- **In-process `@nestjs/schedule` cron in a single web service — no separate worker.**
  The free tier has no always-on background worker and no free cron; one service does webhooks,
  scheduling, and sync. *Cost:* the free web service sleeps after ~15 min idle, so scheduled ticks
  are **missed while it's asleep**. This is survivable, not lost: cursor-based incremental fetches
  *everything changed since the last committed cursor*, so a missed tick only **delays** data.
  Upgrade path = an external pinger (cron-job.org / GitHub Actions) hitting `/internal/sync`, or a
  Render Cron Job.

- **Drizzle ORM over Prisma.**
  Drizzle's `.onConflictDoUpdate({ target, set, where })` compiles to a native, atomic Postgres
  `ON CONFLICT … DO UPDATE` with a version guard — exactly the race-safe write we need under
  concurrent webhook + poll. Prisma's `upsert()` is a non-atomic SELECT-then-write that can race.
  *Cost:* Drizzle is lower-level and more SQL-forward than Prisma's ergonomics.

- **At-least-once delivery + idempotent write = "effectively once".**
  Rather than chase exactly-once delivery, we embrace duplicate/overlapping deliveries and dedup at
  the write: natural key `(source, source_object_type, source_id)` + a version guard on
  `external_updated_at` + a `content_hash` no-op check, with a `webhook_events` ledger to
  short-circuit replays. *Cost:* every record carries a content hash and we keep a webhook ledger.

- **Stripe incremental via the Events API (~30-day retention).**
  The events log is the cleanest "what changed" cursor for Stripe. *Cost:* a cursor older than ~30
  days is purged → `isStaleCursorError` detects it and we fall back to a full object re-list. Built
  in by design.

- **Quarantine bad records and continue the batch, instead of failing the page.**
  A Zod parse failure routes one record to the `quarantine` table and the batch keeps going, so one
  garbage payload can't wedge a source (let alone the run). *Cost:* bad data is parked (and visible
  in `quarantine`) rather than loudly rejected — it needs manual inspection/replay.

- **Soft deletes (`deleted_at`), not hard deletes.**
  Source archive/cancel/refund sets `deleted_at` so history and audit/replay survive. *Cost:*
  read paths must remember to filter out soft-deleted rows.

- **Polling-first for webhooks.**
  HubSpot webhooks require a *public* app (the private-app token is read-only) and Google push
  requires a verified domain; the signature-verified webhook endpoints exist, but the scheduled
  poll is the dependable path. *Cost:* webhook freshness depends on extra provider setup; polling
  cadence (default 15 min) bounds latency otherwise.

- **Pin Node to `20.x` (learned in production).**
  Render initially built on Node 26, where `google-auth-library`'s token refresh failed with
  `Premature close` against the native fetch stack — Stripe/HubSpot were unaffected, which is fault
  isolation working, but Google couldn't sync. Pinning `engines.node` to `20.x` fixed it. *Cost:*
  we're tied to the Node 20 line until the upstream issue clears.

- **Free Render Postgres for the demo.**
  Zero cost, provisioned by `render.yaml`. *Cost:* it's deleted ~30 days after creation — fine for
  a demo, not for production persistence (upgrade the DB plan to keep data).

## Sources & References

- **NestJS** — modules/DI, `@nestjs/schedule`, guards/interceptors: https://docs.nestjs.com
- **Drizzle ORM** — `onConflictDoUpdate` / native upsert: https://orm.drizzle.team/docs/overview
- **pg-boss** — Postgres-backed queue, retries, dead-letter: https://github.com/timgit/pg-boss
- **Stripe API** — Events API & retention, test mode, webhook signatures, Stripe CLI:
  https://docs.stripe.com/api/events , https://docs.stripe.com/webhooks ,
  https://docs.stripe.com/stripe-cli
- **Google Calendar API** — incremental `syncToken`, 410 GONE handling, push `events.watch`:
  https://developers.google.com/calendar/api/guides/sync ,
  https://developers.google.com/calendar/api/guides/push
- **Google OAuth** — refresh tokens via the OAuth Playground:
  https://developers.google.com/oauthplayground
- **HubSpot CRM** — Search API filtered on `hs_lastmodifieddate`, Private Apps, webhooks:
  https://developers.hubspot.com/docs/api/crm/search ,
  https://developers.hubspot.com/docs/api/webhooks
- **Zod** (validation/quarantine boundary): https://zod.dev
- **pino** / **nestjs-pino** (structured logs): https://getpino.io
- **Render** — Blueprints (`render.yaml`), free-tier behavior, Node version:
  https://render.com/docs/blueprint-spec , https://render.com/docs/free
- **AI usage:** this pipeline was designed and built with **Claude (Anthropic)** — planning,
  scaffolding, connector logic, tests, and the live deploy/debug session. Conversation export:
  _<add Claude share link here>_.

## Deploy (Render)

`render.yaml` provisions a free Web Service + free Postgres. Build runs `npm run build`;
start runs migrations then boots (`npm run start:render`). Source credentials are set as
dashboard secrets (`sync: false`). Note: Render free Postgres is deleted ~30 days after
creation — upgrade the plan for persistence.
