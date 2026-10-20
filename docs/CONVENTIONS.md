# CONVENTIONS

## API shape

Base path `/api/v1`. Resources are plural nouns. **No role in the URL.**

```
GET    /api/v1/products?categoryId=&q=&cursor=&limit=
POST   /api/v1/products
GET    /api/v1/products/:id
PATCH  /api/v1/products/:id
DELETE /api/v1/products/:id            → soft archive

POST   /api/v1/orders
GET    /api/v1/orders                  → scoped by actor automatically
POST   /api/v1/orders/:id/cancel       → verb sub-resource for state transitions
POST   /api/v1/credit/:accountId/allocate
```

The same `GET /api/v1/orders` returns the customer their orders, the distributor their sales,
and the admin everything — because scoping happens in the service from `req.ctx.actor`, not
from which file the route lives in.

**Response envelope** — always, including errors:
```jsonc
{ "success": true,  "data": { }, "meta": { "cursor": "...", "hasMore": true } }
{ "success": false, "error": { "code": "ORDER_NOT_FOUND", "message": "...", "details": [] } }
```
Error `code` is a stable SCREAMING_SNAKE string. Panels switch on `code`, never on `message`.

**Pagination** is cursor-based (`cursor` + `limit`, max 100). Offset pagination is only
acceptable for admin tables that need page numbers, and must be stated in the spec.

**Status codes**: 200 read/update · 201 create · 204 delete · 400 validation · 401 no session ·
403 authenticated but not allowed · 404 · 409 conflict/duplicate · 422 business rule failed ·
429 rate limited.

---

## Naming

| Thing | Style | Example |
|---|---|---|
| Files | kebab-case | `order-state-machine.ts` |
| Module files | `<module>.<layer>.ts` | `orders.service.ts` |
| Types / classes / Zod schemas | PascalCase | `CreateOrderBody` |
| Variables, functions | camelCase | `reserveStock` |
| Constants | SCREAMING_SNAKE | `MAX_CART_ITEMS` |
| DB tables (Prisma model) | PascalCase singular | `OrderItem` |
| DB columns | camelCase | `buyerAccountId` |
| Enums | SCREAMING_SNAKE values | `OrderStatus.AWAITING_PAYMENT` |
| Booleans | `is` / `has` / `can` prefix | `isListed` |
| Timestamps | `<verb>At` | `placedAt`, `archivedAt` |
| Money columns | no unit suffix, `Decimal(14,2)` | `grandTotal` |

Never abbreviate: `distributorId` not `distId`, `quantity` not `qty`.

---

## Money

- DB: `Decimal @db.Decimal(14, 2)`. Tax rates `Decimal(5,2)`.
- Code: never do arithmetic with JS `number`. Use `Prisma.Decimal` / `decimal.js` through a
  thin `Money` helper (`Money.add`, `Money.mul`, `Money.pct`, `Money.round`).
- API: money crosses the wire as a **string** (`"1234.50"`), never a float. Panels format it,
  never compute with it — totals come from the server.
- Rounding happens **only on the server**, in `nexa-server/src/core/money.ts`, at the line
  level, half-up. Order total = sum of rounded line totals, so the invoice always adds up.
- Panels never compute money. `src/lib/money.ts` in a panel formats strings for display and
  nothing else — every total, tax, and discount comes from the server already calculated. This
  is why the duplicated-types trade-off is safe: any drift is a display bug, never arithmetic.

---

## Auth

- Access token: JWT, 15 min, `{ sub, role, sessionId }`. Nothing sensitive, nothing encrypted —
  a signed token is not a secret container.
- Refresh token: 30 days, opaque, hashed at rest, **rotated on every use**. Reuse of a rotated
  token revokes the whole session family.
- Cookies: `HttpOnly`, `Secure`, `SameSite=Lax`, `Domain=.nexashopping.in`, one name for all
  roles. The mobile app uses `Authorization: Bearer` instead — same verification path.
- OTP: rate limited per phone *and* per IP, max 5 attempts, 10-minute lock, hashed at rest.
- Every permission check is server-side. Panel UI hiding a button is cosmetic only.

---

## Validation

- One Zod schema per endpoint in `nexa-server/src/schemas/<module>.ts`, covering params, query,
  and body. It must match the entry in `docs/API.md`.
- `validate()` middleware replaces `req` values with the parsed output — downstream code sees
  typed, coerced data.
- Reject unknown keys (`.strict()`) on write endpoints.
- Business rules (stock available, credit sufficient, valid status transition) belong in the
  service, not in Zod. Zod checks shape; services check truth.

---

## Database access

- Repositories return domain objects, not raw Prisma rows with every relation attached.
- `select` explicitly. Never `include` an entire relation tree "just in case" — v1's dashboard
  queries did this and it is why they were slow.
- N+1 is a bug. Batch, or use one query with joins.
- Any write touching two tables: `withTransaction()`. Any read-then-write on money or stock:
  row lock inside the transaction.
- Migrations are additive and reversible where possible: add column → backfill → switch reads →
  drop old column, across separate deploys. Never edit a committed migration.

---

## Frontend (panels)

- Next.js App Router. Server components for data display, client components for interaction.
- **Server state via TanStack Query only.** Redux/Zustand is for genuine client state (open
  modals, filters, wizard steps) — v1 put every API response in Redux, which is why there were
  fourteen near-identical slices per panel.
- All API calls go through `src/lib/api.ts`. No raw `fetch`/`axios` in a component, ever —
  it is the single place auth headers, base URL, and error mapping are handled.
- Response types live in `src/lib/types.ts` only, hand-written from `docs/API.md`.
- Admin and distributor may contain near-identical components. Accepted (ADR-0007). Copy it.
- Forms: react-hook-form + a Zod schema written in the panel from `docs/API.md`.
- Every list screen handles four states explicitly: loading, empty, error, populated.

---

## Testing

| Layer | Tool | Requirement |
|---|---|---|
| Services | Vitest | every branch of pricing, credit, stock, state transitions |
| Endpoints | Supertest + Testcontainers Postgres | happy path + 401 + 403 + 422 |
| Contracts | Vitest | schema round-trips |
| Critical flows | Playwright (later) | login, place order, repay credit |

Non-negotiable tests: order placement (incl. concurrent stock), credit posting (incl.
concurrent), payment webhook idempotency (same event twice = one effect), permission matrix
(each role × each endpoint). Bug fixes ship with a regression test.

---

## Git

- Branches: `feat/orders-placement`, `fix/credit-rounding`, `chore/ci-cache`.
- Conventional commits: `feat(orders): reserve stock on placement`.
- One logical change per PR; if the diff is over ~400 lines, split it.
- PR description states: what, why, schema changes, contract changes, how tested.
- CI must pass lint + typecheck + test + `prisma migrate diff` (no drift) before merge.
- A PR that changes an API shape must state so, and `docs/API.md` must already reflect it.

---

## Security baseline

- Secrets only in env, validated at boot. Never in the repo. Service-account JSON files are
  mounted, never committed.
- Rate limit: auth endpoints hardest, then writes, then reads.
- Uploads: MIME + magic-byte check, size cap, signed direct-to-storage URLs, never trust the
  client's filename.
- PII (Aadhaar, PAN, bank account) encrypted at rest; only last-4 in responses; access logged.
- Structured logs never contain tokens, OTPs, passwords, or full document numbers.
- `helmet`, strict CORS allowlist with credentials, no wildcard origins.
