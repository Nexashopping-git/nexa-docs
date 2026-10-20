# PHASE 0 — Foundation

> **Historical bootstrap checklist:** See [README.md](../README.md) for the active repository map and deployment pointers.

No features. The goal is five repos where the first real feature can be written without any
structural decisions left to make. Expect 3–4 sessions.

Work top to bottom. Each task is one commit. **Task 0 first, before anything else.**

> Claude Code only sees the repo it is opened in. State the repo at the start of every session,
> and keep `nexa-docs` open alongside.

Everything below uses **npm**. No pnpm, no workspaces, no turbo.

---

## Task 0 — Secure the v1 backup

Before creating a single repo:

1. Confirm where the v1 dump lives and that it is somewhere that is not one laptop.
2. Test-restore it into a throwaway Postgres container and count rows on `Account`,
   `Product`, `Order`, `CreditAccount`.
3. Write the location, date, size, and row counts into `DATA-MODEL.md` §10.

A backup nobody has restored is not a backup. We are about to stop caring about v1 — this is
the last moment it's easy to check.

---

## Task 1 — Org and repos

Create under the `nexashopping` org:

```
nexa-docs   nexa-server   nexa-admin   nexa-distributor   nexa-website   nexa-app
```

For each: `main` protected, PR required, CI required to merge.

Push **`nexa-docs` first** — it is the reference every other repo points at. Add an empty
`docs/API.md` with just the section headings; it fills in as endpoints get built.

---

## Task 2 — `nexa-server` skeleton

```bash
mkdir nexa-server && cd nexa-server && npm init -y
npm i express zod prisma @prisma/client decimal.js pino helmet cors cookie-parser
npm i -D typescript tsx vitest supertest @types/node @types/express eslint prettier
npx tsc --init
npx prisma init
```

Then:

1. **Schema** — copy the **identity block only** from `schema/core.prisma`. Catalog and
   inventory land in Phases 2 and 3 as their own migrations. Do not migrate models you aren't
   about to use.
2. **`src/core/`** — nothing else yet:
   - `config.ts` — env parsed with Zod at boot; process exits if anything is missing
   - `errors.ts` — `AppError` hierarchy + the single error-handler middleware
   - `http.ts` — `asyncHandler`, `validate()`, `respond()`, pagination
   - `money.ts` — the `Money` helper, decimal.js, half-up. **Written with its tests, first.**
     Every rupee in the system flows through it.
   - `db.ts` — Prisma client singleton + `withTransaction()`
   - `logger.ts` — pino with requestId
3. **`src/main.ts`** boots and shuts down gracefully. **`src/app.ts`** builds the middleware
   chain and iterates an empty module registry. Add `/healthz` and `/readyz`.
4. `.env.example` committed; `.env` gitignored.

**Verify:** `curl localhost:4000/healthz` → 200. Deleting a required env var makes the process
refuse to start with a readable message.

---

## Task 3 — Local Postgres + Redis

`docker-compose.dev.yml` in `nexa-server` — Postgres 16 + Redis 7 only, pinned versions, named
volumes, health checks. The API runs on the host in development for fast reload.

```bash
docker compose -f docker-compose.dev.yml up -d
npm run db:migrate
```

Scripts in `package.json`: `db:migrate`, `db:studio`, `db:seed`, `db:reset`.

---

## Task 4 — Three Next.js shells

```bash
npx create-next-app@latest nexa-admin --ts --tailwind --app --eslint
```
Same for `nexa-distributor` and `nexa-website`.

In each panel, create the three files that everything else will depend on:

```
src/lib/api.ts      fetch wrapper: base URL from NEXT_PUBLIC_API_URL, auth header,
                    error mapping from the response envelope. The ONLY place fetch is called.
src/lib/types.ts    hand-written response types, mirroring docs/API.md. Empty for now.
src/lib/money.ts    formatting only — never arithmetic.
```

Layout, a placeholder page, no auth, no data yet.

---

## Task 5 — Vercel

Three projects, one per repo. Import the repo, then set a single environment variable:

```
NEXT_PUBLIC_API_URL = https://api.nexashopping.in     (production)
                    = https://api-staging.nexashopping.in   (preview)
```

That is the entire configuration. No tokens, no root directory, no build overrides.

DNS: `api.` → the VPS, `admin.` and `distributor.` → Vercel, apex → website.

---

## Task 6 — CORS, end to end

The first thing that will actually break. In `nexa-server`, an explicit origin allowlist with
credentials enabled — no wildcards:

```
https://admin.nexashopping.in
https://distributor.nexashopping.in
https://nexashopping.in
http://localhost:3000  (dev only)
```

Cookies: `HttpOnly`, `Secure`, `SameSite=Lax`, `Domain=.nexashopping.in` so a Vercel-hosted
panel can hold a session issued by the VPS.

**Verify before moving on:** a deployed panel successfully calls `/healthz` on the deployed API
from the browser. Prove this now, not in Phase 1 during auth.

---

## Task 7 — CI, in every repo

GitHub Actions on every PR: `npm ci` → lint → typecheck → test. `nexa-server` adds
`prisma migrate diff` against a service-container Postgres.

---

## Task 8 — Guardrails

The rules in `CLAUDE.md` are only real if they're enforced. In `nexa-server`:

- ESLint `no-restricted-imports`: Prisma may only be imported from `*.repository.ts`,
  `core/db.ts`, and seeds.
- ESLint ban on `parseFloat`/`Number()` applied to anything typed as money.
- A test that fails if any Prisma model has a `Float` field.
- A test that fails if any `@relation` declares `onDelete: SetNull` on a required field —
  the exact v1 bug, caught mechanically.
- `lint-staged` + husky, conventional-commit check.

In each panel: ESLint rule banning raw `fetch`/`axios` outside `src/lib/api.ts`.

---

## Task 9 — Production Docker

In `nexa-server`: `Dockerfile` (multi-stage, non-root user, `npm ci --omit=dev` in the runtime
stage) and `docker-compose.yml` with api + postgres + redis + Caddy. A deploy script that runs
migrations as an explicit step *before* swapping containers. Nightly `pg_dump` to object
storage.

Do not point production DNS at it yet — this task just proves the image builds and runs.

---

## Task 10 — Per-repo `CLAUDE.md`

Each repo gets a short one: what this repo is, that `nexa-docs` is the source of truth
(including `API.md`), the repo-specific rules, and the commands. Twenty lines, not a copy of
the full instruction set — a forked copy will drift and then contradict the original.

---

## Exit criteria

- [ ] v1 backup verified and documented
- [ ] Five repos created, protected, CI green in each
- [ ] `Money` helper tested to the paisa
- [ ] Server boots against dockerised Postgres; `/healthz` and `/readyz` return 200
- [ ] Identity migration applied, no drift
- [ ] Guardrail tests actually fail when you deliberately introduce a `Float` money field
- [ ] Three Vercel projects deployed, **and one of them proven to call the live API from a
      browser without CORS errors**
- [ ] `docs/API.md` exists with its structure in place
- [ ] Every repo has its own `CLAUDE.md` pointing at `nexa-docs`

Then start Phase 1.

---

## Before Phase 4

Q9 (GST invoicing) shapes the order money breakdown, and Q4/Q5 shape credit and payments.
None block Phases 0–3, but answer them while catalog and inventory are being built — not during
order placement.
