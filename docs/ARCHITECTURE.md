# ARCHITECTURE

> **Historical design note (2026-10-20):** This document records the original architecture. See [the repository overview](../README.md) for active repositories and deployments. The production API Compose stack currently runs PostgreSQL, API, and Caddy without Redis; the separate Vite diary portal is hosted on Cloudflare Workers.

## 1. Guiding idea

v1 modelled **roles** as first-class structure — separate routes, separate tables, separate
code paths for admin / distributor / user. That is why the same logic exists three times.

v2 models **capabilities on shared entities**. There is one product catalog, one inventory
concept, one order concept. Who may do what is a runtime permission check, not a directory.

The second idea: **the domain is financial**. Credit, orders, and payments must be
reconstructible from an append-only history. Anywhere state can drift from its history is a
bug waiting to be a support ticket.

---

## 2. Stack

| Concern | Choice | Why |
|---|---|---|
| Language | TypeScript, strict, ESM | shared types across server + panels |
| Server | Node 22 + Express 5 | team already knows it; keep the boring part boring |
| ORM | Prisma | already in use, migrations are sound |
| DB | PostgreSQL 16 | relational + transactional; the domain is money |
| Cache / locks | Redis (deferred) | add before payments/jobs need distributed coordination |
| Auth limits | PostgreSQL fixed-window buckets | current low-volume OTP abuse protection; subjects are hashed |
| Validation | Zod, in `nexa-server/src/schemas/` | one schema drives runtime validation + types |
| Panels | Next.js (App Router) + Tailwind + TanStack Query | Vercel-native, no SPA auth gymnastics |
| Repos | five independent repos under the org | independent deploys, no build coupling |
| Package manager | npm | no workspaces needed, so nothing more is warranted |
| Sharing | none — `docs/API.md` is the contract | a package is not worth its overhead here |
| Tests | Vitest + Supertest + Testcontainers | fast unit, real-Postgres integration |
| Logs / metrics | pino + prom-client + OpenTelemetry-ready | v1 already exported Prometheus metrics |

Panel stack is settled: Next.js App Router, three Vercel projects, subdomains (ADR-0004).
Express vs Fastify stays Express unless a benchmark says otherwise — recorded if we change it.

---

## 3. Request lifecycle

```
request
  → requestId + logger context
  → helmet, cors, body parse, cookie parse
  → rateLimit(bucket)
  → authenticate()          resolves session → req.ctx.actor {accountId, role, permissions}
  → validate(schema)        params / query / body, from src/schemas/
  → authorize('perm')       coarse: does this role hold the permission at all
  → controller              thin
      → service             business rules; calls policy.canX(actor, resource) for
      │                     row-level checks (e.g. "is this your order?")
      → repository          prisma only
  → respond()               uniform envelope
  → errorHandler            AppError → status + code; anything else → 500 + logged
```

Two-stage authorization is deliberate:
- **`authorize('order:read:any')`** — route-level, cheap, does the role even have this power.
- **`policy.canReadOrder(actor, order)`** — after loading the row: ownership, distributor
  scoping, status gates. This is what v1 did with scattered `if (role === 'ADMIN')`.

### Permission model

Permissions are strings: `resource:action[:scope]`, e.g.
`product:write`, `order:read:own`, `order:read:any`, `credit:allocate`, `kyc:approve`.

One table maps role → permission set, defined in code (`core/auth/permissions.ts`), not in the
database, so it is reviewable in a diff. Co-admin is a role with a subset — not a separate
code path.

---

## 4. Modules

Each is a folder under `nexa-server/src/modules/`, registered once in `app.ts`:

| Module | Owns |
|---|---|
| `auth` | OTP/Firebase verification, session issue + rotation, logout, device tokens |
| `accounts` | account CRUD, role assignment, profile, distributor onboarding, referrals |
| `kyc` | document submission, review workflow, status transitions |
| `catalog` | categories, attributes, products, variants, media |
| `inventory` | stock per owner, ledger, price lists, low-stock alerts |
| `cart` | cart lifecycle, line items, re-pricing at read time |
| `orders` | placement, state machine, fulfilment, cancellation, returns |
| `payments` | gateway sessions, callbacks, idempotency, refunds, reconciliation |
| `credit` | limits, ledger, allocation, repayment, exposure reporting |
| `catalogFeedback` | ratings + reviews |
| `notifications` | templates, fan-out, device tokens, read state |
| `content` | banners, static content |
| `reporting` | dashboards, aggregates, exports |

Registry pattern replaces v1's 60 imports:

```ts
// app.ts
const modules = [authModule, accountsModule, kycModule, catalogModule, /* ... */];
modules.forEach(m => app.use(`/api/v1/${m.basePath}`, m.router));
```

---

## 5. Keeping server and panels in agreement

There is no shared package. `nexa-docs/API.md` is the contract, maintained by hand.

**Why this is acceptable here.** A published contract package buys compile-time enforcement,
paid for with a registry, tokens, version bumps, and a three-repo dance for every change. With
one developer and Claude Code — which reads `API.md` in whichever repo it is opened in — the
document does the same job as the discipline, without the machinery.

**What it costs.** Nothing mechanically catches a drift. A server response that gains a field
the panel does not know about is harmless; one that renames or removes a field breaks the panel
silently at runtime. So:

- `API.md` is updated in the **same session** as the server change, never after.
- Renaming or removing a response field is treated as a breaking change: find every panel that
  reads it before merging. `grep` for the field name across the panel repos.
- Adding fields is always safe. Prefer adding over renaming.
- Panels type API responses in `src/lib/types.ts` and nowhere else, so there is exactly one
  place to update per repo.

**If this ever stops working** — more than one or two silent-drift incidents, or a second
developer joins — that is the signal to introduce a shared contract package. Record it as an
ADR; do not do it pre-emptively.

**Database access** stays server-only. Panels reach data through the API. Only `nexa-server`
has a Prisma dependency.

## 6. Cross-cutting rules

**Errors** — one hierarchy, one handler:
```
AppError(code, httpStatus, message, details?)
├── ValidationError 400   ├── UnauthorizedError 401  ├── ForbiddenError 403
├── NotFoundError 404     ├── ConflictError 409      ├── UnprocessableError 422
├── RateLimitError 429    └── ExternalServiceError 502
```
Services throw. Controllers never catch. The handler logs and maps.

**Idempotency** — mandatory on: payment initiation, gateway callbacks, order placement,
credit allocation, repayment. Redis key + a `ProcessedEvent` table for gateway callbacks so a
retried PhonePe webhook can never double-credit an account. v1 had none of this.

**Transactions** — any operation crossing tables uses `withTransaction()`. Order placement is
the canonical example: reserve stock → write inventory ledger → create order + items →
debit credit ledger → create payment intent. All or nothing.

**Caching** — Redis, with an explicit key registry and explicit invalidation on write. Only
for: product listings, category tree, banners, dashboard aggregates. Never for anything
financial. Never cache without a documented invalidation trigger.

**Background jobs** — order expiry, low-stock alerts, notification fan-out, reconciliation,
report generation. Not `node-cron` inside the web process (v1 did this, so jobs ran once per
replica and blocked scaling). Either a separate worker container or BullMQ on Redis.
> `[DECISION NEEDED]` see Q7. Not blocking until Phase 4.

**Observability** — structured JSON logs with requestId + accountId, never logging tokens,
OTPs, card data, or KYC numbers. `/healthz` (liveness), `/readyz` (currently DB; Redis once
introduced), `/metrics`.

---

## 7. Deployment

**Server (Docker on VPS)**
```
Current: docker-compose.yml:  api  ·  postgres  ·  caddy (TLS)
Later phases:                 redis  ·  worker  ·  additional api replicas as needed
```
- Multi-stage Dockerfile, non-root user, `npm ci --omit=dev` in the runtime stage.
- Caddy instead of nginx+certbot: automatic TLS, one config file. v1's certbot renew loop with
  a mounted docker socket is a security smell we are not carrying over.
- Migrations run as an explicit step in the deploy script, never on container start.
- Nightly `pg_dump` to object storage with a documented restore drill.

**Panels + site (Vercel)**
- Three Vercel projects, each connected to its own repo. Import the repo, set one environment
  variable, deploy. No registry tokens, no root directory, no build config.
- `NEXT_PUBLIC_API_URL` per environment; preview deployments point at staging.
- Auth cookies: `Domain=.nexashopping.in`, `SameSite=Lax`, `Secure`, `HttpOnly` — so
  `admin.nexashopping.in` on Vercel can talk to `api.nexashopping.in` on the VPS.
  CORS allowlist is explicit, credentials on, no wildcards.

```
admin.nexashopping.in       → Vercel ─┐
distributor.nexashopping.in → Vercel ─┼→ api.nexashopping.in → Caddy → api containers
nexashopping.in             → Vercel ─┘                                  ├→ postgres
mobile app ─────────────────────────────────────────────────────────────┘└→ redis
```

The topology diagram above includes the planned Redis connection; Redis and the worker are not
part of the current deployment described in `nexa-server/docker-compose.yml`.

**Environments** — local (docker compose, dev DB) · staging (branch deploys, seeded) ·
production. Env vars parsed with Zod at boot; the process refuses to start if any is missing.

**Release coordination.** Deploy the API first; it stays backward-compatible so panels can
deploy on their own schedule. Adding response fields never requires a coordinated deploy.
Renaming or removing one does — flag it in the PR description and deploy the panel in the same
window.
