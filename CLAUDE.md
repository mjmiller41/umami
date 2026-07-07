# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A **fork of [Umami](https://github.com/umami-software/umami)** (v3.2.0) — a self-hosted, privacy-focused
web-analytics platform — run to track traffic across the domains Michael owns. `origin` is the fork
(`git@github.com:mjmiller41/umami.git`); upstream is `umami-software/umami`. This is **operating and lightly
customizing an upstream product**, not building one from scratch. Deploys to a **Hostinger** server backed by
**PostgreSQL**.

## Fork discipline (read before editing)

- **Keep changes surgical and merge-friendly.** Every local edit is a future merge conflict when pulling
  upstream. Prefer configuration (env vars) and additive files over modifying upstream source. Touch upstream
  files only when there's no config path to the outcome.
- **Sync via a second remote.** Add upstream once (`git remote add upstream https://github.com/umami-software/umami.git`),
  then update with `git fetch upstream && git merge upstream/master` (or rebase local work). Don't let the fork drift.
- **Record why for any upstream-file change** in the commit body, so the next merge knows whether to keep it.
- Branch per task (`feature/…`, `fix/…`); `main` stays deployable.

## Stack

- **Next.js 16** (App Router) + **React 19** + **TypeScript**, bundled with Turbopack.
- **Prisma 7** over **PostgreSQL** (via `@prisma/adapter-pg`) — the primary/relational store. `scripts/check-db.js`
  enforces Postgres **9.4+** (upstream docs recommend 12.14+); this version has **no MySQL support** (schema is Postgres-only).
- **ClickHouse** (optional) — columnar store for analytics events at scale; **Redis** (optional) caching;
  **Kafka** (optional) event ingestion. All three are off unless their `*_URL` env var is set.
- **pnpm** workspaces. **Biome** for lint/format (not ESLint/Prettier). **Vitest** unit tests, **Playwright** e2e.
- UI: `@umami/react-zen` + `react-aria-components`, Chart.js, `rrweb` (session replay), `react-simple-maps`.

## Commands

```bash
pnpm install
pnpm dev                 # dev server (Turbopack), reads .env via dotenv
pnpm build               # full pipeline: check-env → build-db → check-db → build-tracker → -recorder → -geo → -app
pnpm start               # serve production build on :3000 (put a reverse proxy in front)

pnpm test                # Vitest once   ·  pnpm test:watch  ·  pnpm test:e2e (Playwright)
pnpm vitest run <file>   # single unit test file
pnpm check               # Biome lint+format autofix  ·  pnpm lint  ·  pnpm format

pnpm update-db           # prisma migrate deploy (apply migrations to an existing DB)
pnpm change-password     # reset a user's password
pnpm seed-data           # seed sample analytics data
```

First `pnpm build` against an empty DB creates the tables **and a default login `admin` / `umami` — change it
immediately.** Requires `DATABASE_URL` in `.env` (`check-env` fails the build without it).

## Architecture — the big picture

Two data paths meet at one storage layer:

1. **Collection (public, unauthenticated).** The tracker script `src/tracker/index.js` (built by
   `rollup.tracker.config.js` → served as `/script.js`) is embedded on each tracked domain. It POSTs events to
   **`src/app/api/send/route.ts`**; the `src/app/(collect)/p/[slug]` and `q/[slug]` routes plus `docker/proxy.ts`
   are the alternate collection/proxy entry points. Session-replay events go through the `recorder` + `record` API.
2. **Dashboard (authenticated).** The `src/app/(main)` App-Router UI reads through `src/app/api/*` route handlers.

**The DB routing spine is `src/lib/db.ts` → `runQuery({ prisma, clickhouse, kafka })`** (fan-in ~71). Every
data operation supplies per-backend implementations and `runQuery` dispatches by env: ClickHouse if
`CLICKHOUSE_URL` is set (Kafka variant when producing), otherwise Prisma/Postgres. Because of this, analytics
queries are written **twice** and must stay in sync:
- `src/queries/prisma/*` — relational/config entities (users, teams, websites, reports, boards) via Prisma.
- `src/queries/sql/*` — analytics aggregations (pageviews, sessions, events, performance, heatmap, replays),
  hand-written SQL that runs on **both** Postgres and ClickHouse (see `src/lib/clickhouse.ts` `rawQuery`/`parseFilters`).

**Request pipeline** (the top hotspots trace it): API route → `parseRequest` (`src/lib/request.ts`, validates
with Zod schema + auth) → `src/permissions/*` capability check → query layer → `json()`/`unauthorized()`
(`src/lib/response.ts`). Auth is JWT (`src/lib/jwt.ts`, `src/lib/auth.ts`); `APP_SECRET` signs tokens.

**Layers** (enforced by call direction): `app` (entry) → `lib` (core utilities, highest fan-in) + `components`
+ `queries`; `permissions` gates entity access; `store` (Zustand) + `components/hooks` back the UI. i18n runs
through `useMessages` (`next-intl`) — the single highest-fan-in symbol; never hardcode UI strings, add message keys.

## Repo layout

```
src/tracker/            # embeddable collection script → built to /script.js
src/app/(collect)/      # public event-collection routes (p/[slug], q/[slug])
src/app/api/            # authenticated route handlers (send, websites, reports, teams, realtime, record, …)
src/app/(main)/         # dashboard UI (App Router)
src/lib/db.ts           # runQuery — dispatches each query to prisma | clickhouse | kafka
src/queries/prisma/     # relational entity queries (Postgres via Prisma)
src/queries/sql/        # analytics SQL, runs on Postgres AND ClickHouse — keep the two backends in sync
src/permissions/        # per-entity access checks, called from API routes
src/lib/                # core: request/response, auth/jwt, clickhouse, redis, kafka, schema (Zod), formatting
src/components/          # UI: hooks/, charts/, metrics/, input/, common/
prisma/schema.prisma    # Postgres schema + prisma/migrations/
db/clickhouse/schema.sql # ClickHouse DDL (only when running the ClickHouse backend)
```

## Configuration (env)

`.env` at repo root. Core: `DATABASE_URL` (Postgres), `APP_SECRET` (token signing). Deploy path:
`BASE_PATH` (serve under a sub-path). Optional backends: `CLICKHOUSE_URL`, `REDIS_URL`, plus Kafka vars.
Tracker: `TRACKER_SCRIPT_NAME` / `TRACKER_SCRIPT_URL` (rename/relocate `script.js` to dodge ad-blockers),
`COLLECT_API_ENDPOINT`. `SKIP_DB_CHECK` bypasses the pre-build DB version check. See `scripts/check-env.js`
and `next.config.ts` for the full surface.

## Deployment (Hostinger app + Neon Postgres)

The app is a long-running Node process (Node **18.18+**), not static — it runs on **Hostinger** (Node.js app via
GitHub connection). The **database is not on Hostinger**: shared hosting only offers MySQL, and this version
dropped MySQL, so Postgres is provided by **Neon** (managed, free tier).

- **Database:** Neon project `neon-aqua-island` (`red-band-98389483`), dedicated database **`umami`** (isolated from
  the pre-existing `neondb` app in the same project — Umami's `user`/`session` tables would otherwise collide).
  Connection uses the **direct** (non-pooler) endpoint. `DATABASE_URL` + `APP_SECRET` live in `.env` (gitignored)
  locally and must be set as env vars in the Hostinger app panel. Neon is driveable via the Neon MCP server.
- **Deploy:** Hostinger builds from the GitHub repo — build `pnpm install && pnpm build`, start `pnpm start`
  (`next start`, honors `PORT`). First build runs migrations and seeds the default `admin`/`umami` login
  (already done against Neon — **change that password**). Re-apply migrations later with `pnpm update-db`.
- **Watch:** `next build --turbo` is memory-hungry; if shared hosting OOMs the build, build in CI/Docker and ship
  the output, or use the `Dockerfile`/`docker-compose.yml`. Hostinger runners may need `corepack enable` (or
  `npm i -g pnpm`) since the repo is pnpm-based.

## Owner

Michael J. Miller — solo full-stack (Node, PHP/Laravel, TypeScript, React), Avon Park FL. Prefers concise,
direct communication. This runs alongside his other projects (LaserReady, LaserKerf); keep it independent.
