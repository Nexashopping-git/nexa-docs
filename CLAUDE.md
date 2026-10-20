# CLAUDE.md — NexaShopping v2 Working Agreement

> **Current-state note (2026-10-20):** Read [README.md](README.md) first. This file captures the original August working agreement; several implementation and deployment details below are historical. In particular, the production Compose stack has no Redis, the diary portal is a sixth application repository, and the current API origin is `https://api.nexashopping.in`. Verify behavior against application code and current API documentation before treating an older plan or example as implemented.

> Read this file at the start of **every** session, along with `docs/ROADMAP.md`
> (to know which phase we are in) and the spec for the feature being built.
> If anything in this file conflicts with what you are about to do, stop and ask.

---

## 1. What we are building

NexaShopping is a **B2B2C credit-based distribution platform**.

- **Admin** runs the central catalog and holds central stock. Issues credit limits to distributors.
- **Distributor** buys from admin (often on credit), holds their own stock, and sells to
  end customers — including creating orders/bills on a customer's behalf.
- **Customer** buys through the mobile app.
- **Repayment** flows back up: distributor repays admin through the payment gateway.

Surfaces:
| Surface | Stack | Deployment |
|---|---|---|
| API server | Node + TypeScript + Prisma + PostgreSQL + Redis | Docker (VPS) |
| Admin panel | Next.js | Vercel |
| Distributor panel | Next.js | Vercel |
| Marketing site | Next.js (landing + policy pages) | Vercel |
| Mobile app | existing app, consumes the same API | store |

**Only the server is dockerized.** Panels and site deploy to Vercel with zero infra config.

---

## 2. The ten golden rules

1. **One resource, one route file.** Never split routes by role. Access is decided by a
   permission check, not by a URL prefix. `/api/v1/products`, not `/admin/api/v1/product`.
2. **Controllers are thin.** Parse → call service → format response. No Prisma, no business
   logic, no `if (role === ...)` in a controller. Target: under 20 lines per handler.
3. **Prisma is only touched inside `repositories/`.** Services call repositories. Nothing else
   imports `prisma`.
4. **Money is never a `Float`.** See `docs/CONVENTIONS.md` §Money. One representation, everywhere.
5. **Financial state is a ledger, never a mutable column.** Balances are derived or maintained
   inside a transaction that also writes the ledger row. Never one without the other.
6. **Order and payment records are immutable snapshots.** An order stores what was true at
   purchase time — address text, product name, unit price, tax — not a FK to a mutable row.
7. **Every write that touches two tables runs in `prisma.$transaction`.** No exceptions.
8. **`docs/API.md` is the contract.** Server and panels agree because both were written from
   that file, not because a package enforces it. Change the API → update `API.md` in the same
   session, before or alongside the code. Never after.
9. **No new dependency without asking.** Same for any new top-level directory.
10. **If the spec is ambiguous, ask — do not guess and do not scaffold "flexibility".**
    Speculative abstraction is how v1 got to 11,000 lines.

---

## 3. Repo layout

Five independent repositories under the `nexashopping` GitHub org. **No shared packages, no
monorepo, no publishing.** Each repo stands alone and installs with `npm`.

```
github.com/nexashopping/
├── nexa-docs           # THIS instruction set — the contract lives here too (docs/API.md)
├── nexa-server         # API — the only dockerized repo → VPS
├── nexa-admin          # Next.js → Vercel
├── nexa-distributor    # Next.js → Vercel
├── nexa-website        # Next.js → Vercel (landing + legal)
└── nexa-app            # mobile
```

**How the panels reach the server:** one environment variable.
`NEXT_PUBLIC_API_URL=https://api.nexashopping.in`. Deploying a panel is: import the repo into
Vercel, set that variable, done.

**How types stay in sync:** by hand, against `nexa-docs/API.md`. There is no package enforcing
it. See §4a — this is the one discipline that carries real weight.

### `nexa-server`
```
prisma/                  schema.prisma, migrations/, seed.ts
src/main.ts              boot only: env check → server → graceful shutdown
src/app.ts               express instance, middleware chain, module registry
src/modules/<module>/    routes · controller · service · repository · policy · test
src/core/                config · errors · http · auth · db · cache · queue · observability
src/integrations/        phonepe · firebase · storage · sms — each behind an interface
src/schemas/             Zod schemas, one file per module — mirrors docs/API.md
Dockerfile · docker-compose.dev.yml
```
A module may import from `core/` and its own folder. **A module must not import another
module's repository.** Cross-module needs go through the other module's *service*.

### `nexa-admin` / `nexa-distributor` / `nexa-website` — identical layout, so context transfers
```
src/app/(auth)/…  (dashboard)/…      App Router route groups
src/features/<domain>/                components · hooks
src/lib/api.ts                        the ONLY place fetch is called
src/lib/types.ts                      hand-written types, mirrors docs/API.md
src/lib/money.ts                      formatting only — never arithmetic
src/components/                       this app's components
```

Admin and distributor will contain similar components. **That duplication is accepted on
purpose** — copying a Button is cheaper than a shared package. Never introduce one without an
ADR.

## 4. How a session works

**Start of session** — say this back to me in one line before writing code:
> "Repo R, phase N, task X. Touching modules A and B. API changes: yes/no."

If the answer is yes, open `nexa-docs/API.md` first.

Then:
1. Read `docs/ROADMAP.md` → confirm the current phase and the exact task.
2. Read the feature spec in `docs/features/`. If none exists, write it first and get approval.
3. Read the existing code in the modules you'll touch. Do not assume — the file may already
   have the helper you were about to write.

**During**
- Contract first: update `docs/API.md`, then the server schema, then the panel types.
- One task per commit. If a task grows past ~400 changed lines, stop and split it.
- Write the test in the same commit as the code, not "later".
- Update the doc in the same commit if behaviour changed.

### 4a. Changing the API (the one discipline that matters)

Nothing mechanically stops the server and a panel from disagreeing. `docs/API.md` is what
prevents it, and it only works if it is never stale.

When an endpoint's request or response shape changes:

1. Update the endpoint's entry in `nexa-docs/API.md`.
2. Update the Zod schema in `nexa-server/src/schemas/`.
3. Update `src/lib/types.ts` in every panel that calls it.

All three, same session. A change that lands in the server but not in `API.md` is a bug that
surfaces weeks later as a panel silently rendering `undefined`.

If you are working in a panel and `API.md` does not describe the endpoint you need — **stop and
say so.** Do not guess the response shape from what seems reasonable.

**End of session** — output a short handoff block:
```
REPO:      <which repo>
DONE:      <what merged>
API.md:    <updated / not needed>
DECIDED:   <any decision made, so it can go into nexa-docs/DECISIONS.md>
OPEN:      <anything left half-finished, with file paths>
NEXT:      <the single next task, and in which repo>
```

---

## 5. Definition of done

A task is done only when all of these are true:

- [ ] Input validated with a Zod schema in `src/schemas/`, matching `docs/API.md`
- [ ] Authorization enforced by a policy check (not by which route file it lives in)
- [ ] Errors thrown as `AppError` subclasses; no bare `res.status(500)`
- [ ] Multi-table writes wrapped in a transaction
- [ ] Money handled per `docs/CONVENTIONS.md`; no `Float`, no `parseFloat` on money
- [ ] Every new FK has an index and an explicit, *legal* `onDelete`
- [ ] Unit test for service logic + integration test for the endpoint
- [ ] `npm run lint && npm run typecheck && npm test` pass
- [ ] Docs updated if the contract, schema, or a decision changed

---

## 6. Anti-patterns — these are the v1 mistakes, do not repeat them

Concrete list, drawn from the old repo. If a diff introduces one of these, reject it.

| ❌ Never | ✅ Instead |
|---|---|
| `routes/product/admin.routes.ts` + `distributor.routes.ts` + `user.routes.ts` | one `products.routes.ts` + `can('product:write')` |
| 2,000-line `authController.ts` | auth module split into service / session / policy |
| `AdminInventory` and `DistributorInventory` as separate tables | one `Inventory` keyed by stock owner |
| `AdminInventoryLog` and `DistributorInventoryLog` | one append-only `InventoryLedger` |
| `userId` + `userType` polymorphic pairs on Order | explicit `buyerAccountId` / `sellerAccountId` FKs |
| `referredBy String` with no FK, *and* a `Referral` table | one referral edge, one source of truth |
| `price Float` next to `salePrice Decimal` | one money type, project-wide |
| `availableCredit` written directly | derived from / updated atomically with the ledger |
| `onDelete: SetNull` on a non-nullable column | nullable + SetNull, or Restrict, and say why |
| `Category` cascade-deleting `Product` | `Restrict` + soft-deactivate |
| `Order.addressId` FK for a shipped order | address snapshot copied onto the order |
| `@@unique([accountId, isActive])` to fake "one active cart" | proper cart lifecycle with a status enum |
| 60 hand-written route imports in `index.ts` | a module registry array |
| RSA-encrypting the JWT payload | standard signed JWT; put nothing secret in it |
| Cookies named `at` / `dt` / `ut` per role | one session cookie; role comes from the session |
| Redux slice per feature duplicated in admin *and* distributor | TanStack Query + one `src/lib/api.ts` |
| Commented-out seed/cleanup calls left in `index.ts` | CLI scripts in `nexa-server/scripts` |
| Webhook handler with no idempotency | idempotency key + dedupe table on every gateway callback |

---

## 7. Commands

**npm everywhere.** No pnpm, no workspaces, no turbo.

Every repo:
```bash
npm install
npm run dev
npm run lint && npm run typecheck && npm test
```

`nexa-server` also:
```bash
npm run db:migrate · db:studio · db:seed · db:reset
docker compose -f docker-compose.dev.yml up -d      # postgres + redis
```

## 8. Document index

| File | Purpose |
|---|---|
| `docs/ARCHITECTURE.md` | system design, module boundaries, request lifecycle, deployment |
| `docs/DATA-MODEL.md` | schema redesign and the reasoning behind every change |
| `docs/CONVENTIONS.md` | naming, API shape, errors, money, auth, caching, testing |
| `docs/ROADMAP.md` | phased build order with exit criteria |
| `docs/API.md` | **the endpoint contract** — every request/response shape |
| `docs/PHASE-0.md` | the bootstrap task list — start here |
| `docs/schema/core.prisma` | reviewed schema for identity, catalog, inventory |
| `docs/DECISIONS.md` | ADR log — every non-obvious choice, dated |
| `docs/OPEN-QUESTIONS.md` | unresolved decisions; **blocks work when marked BLOCKING** |
| `docs/features/NN-name.md` | per-feature spec, written before the feature |

These live in **`nexa-docs`**, along with `docs/API.md` — the endpoint contract. Every other
repo carries a short `CLAUDE.md` that points here and adds only its own repo-specific rules.
When a rule changes, it changes in `nexa-docs` — never forked into a repo copy.

Anything marked `[DECISION NEEDED]` in any doc means: **do not implement around it, ask.**
