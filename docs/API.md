# API CONTRACT

> **Reconciliation note (2026-10-20):** The diary section below was checked against `nexa-server` routes, schemas, service, and controller. Older sections were written during earlier phases and have not all been re-audited. For customer mobile endpoints, also consult the server's `docs/openapi/customer-mobile.yaml` and current route code.

**This file is the agreement between `nexa-server` and the panels.** There is no shared
package. Both sides are written from this document, so it is only useful if it is never stale.

Rules:
- Update this file in the **same session** as the server change. Never after.
- Adding a response field is safe. **Renaming or removing one is breaking** — grep the panel
  repos for the field name before merging.
- If an endpoint you need is not described here, it does not exist yet. Do not guess its shape.

Status legend: 🟢 live · 🟡 in progress · ⚪ planned

---

## Conventions

Base URL: `https://api.nexashopping.in/api/v1`

The customer mobile team's canonical OpenAPI 3.1 contract is tracked in
`nexa-server/docs/openapi/customer-mobile.yaml`. It contains the production URL only and is
validated against the customer route manifest by `npm run openapi:validate`.

**Every response uses this envelope**, success or failure:

```jsonc
// success
{ "success": true, "data": { }, "meta": { "cursor": "abc", "hasMore": true } }

// failure
{ "success": false, "error": { "code": "ORDER_NOT_FOUND", "message": "…", "details": [] } }
```

Panels switch on `error.code`, never on `error.message`.

**Money** is always a **string** (`"1234.50"`), never a number. Panels format it; they never
compute with it. Every total, tax, and discount arrives already calculated.

**Timestamps** are ISO 8601 UTC strings. **IDs** are UUID strings.

**Pagination** varies by endpoint. Older cursor endpoints use `?cursor=<opaque>&limit=<1-100>`
with `meta.cursor` and `meta.hasMore`. Diary endpoints use `?page=<positive integer>&limit=<1-100>`
with `meta: { page, limit, total, totalPages }`.

**Auth**: `Authorization: Bearer <token>` for every client (web panels and mobile). The token
is an **opaque** session token — the server stores only its SHA-256 hash, and it maps to a row
in the `Session` table so it can be revoked. (Cookies were dropped: panels on Vercel and the API
on a different host make cross-site cookies unreliable.)

**Common error codes**: `VALIDATION_FAILED` 400 · `UNAUTHENTICATED` 401 · `FORBIDDEN` 403 ·
`NOT_FOUND` 404 · `CONFLICT` 409 · `BUSINESS_RULE_FAILED` 422 · `RATE_LIMITED` 429 ·
`INTERNAL` 500

---

## Endpoint template

Copy this block for every new endpoint. Fill it in before writing the handler.

````
### `METHOD /path` ⚪
Who: ADMIN · DISTRIBUTOR · CUSTOMER        Permission: `resource:action`

Request
```ts
{ }
```

Response `data`
```ts
{ }
```

Errors: `CODE` — when it happens
Notes: scoping, side effects, idempotency
````

---

## Auth  *(Phase 1)*

Admin signs in with **Google** (Firebase). Distributors and customers sign in with a
DLT-approved Ping4SMS phone OTP. Every implemented path ends with a `Session` row and an
opaque bearer token.

### `POST /auth/customer/otp/request` live
Who: public (pre-auth)

Request: `{ phone: string, referralCode?: string }`. The phone is E.164. The optional
seven-character referral code is considered only when registering a new customer. A valid
code creates immutable `REFERRAL` acquisition under that distributor; no code creates
immutable `ORGANIC` acquisition under the primary admin. Response: `{ ok: true }`.

### `POST /auth/customer/otp/verify` live
Who: public (pre-auth)

Request: `{ phone: string, otp: string }`. Response matches the other session-issuing login
endpoints. A new customer account, profile, acquisition relationship, session, and OTP
consumption are committed atomically only after a valid OTP. OTPs expire and are single-use.

### Customer mobile resources live

The implemented mobile surface is `/customer/profile`, `/customer/addresses`,
`/storefront/products`, `/customer/cart`, `/customer/cart/quote`, and `/customer/orders`.
The OpenAPI file is authoritative for exact requests, response envelopes, examples,
pagination, and errors. The storefront never exposes or asks for a distributor. A quote uses
the attributed distributor only when it can fulfil every line; otherwise the entire cart goes
to admin. At placement, lost distributor capacity causes whole-order admin fallback at the
locked quote prices. Orders are never split.

`GET/POST/PATCH/DELETE /customer/cart[...]` all return the same `CartView`, each item
carrying a live, unlocked `offer: { price, discountPrice, available }` — the same
preferred-distributor-else-admin resolution the storefront listing uses (`price` is the
distributor's `sellPrice` when they currently have stock, else admin's `customerPrice`;
`null`/`0` if neither lists the variant). This is a *display* price only, re-evaluated on
every cart read; it can drift right up until `POST /customer/cart/quote` locks it for
checkout. Each item also carries `image: { url, alt } | null` (the variant's own image if
set, else the product's primary image).

**Client rendering rule for `offer.price` / `offer.discountPrice`** — matches the money math
`core/pricing.ts` uses to lock the quote (it treats `sellPrice` as the gross list price and
subtracts `sellPrice - discountPrice` as the line's discount before tax), so this is what the
customer is actually charged, not a cosmetic promo badge:
- **Amount actually charged per unit = `discountPrice ?? price`.**
- `discountPrice` is **only ever non-null on a distributor-fulfilled line** — admin pricing
  (direct customers, or a line that fell back to admin) never sets it. So it resolves *per
  line*, not cart-wide: one line can show a distributor discount while another in the same
  cart shows plain admin pricing with `discountPrice: null`.
- Render: if `discountPrice !== null`, show `price` struck-through ("was") and
  `discountPrice` as the highlighted sale price ("now"). If `discountPrice === null`, show
  `price` plain — there is nothing to discount against.
- `available` is stock count, not pricing — use it for "Only N left" copy / disabling
  "add" past that quantity (also now enforced server-side, see `POST .../items` below), never
  folded into the price line.

- `GET /customer/cart` — the current cart.
- `POST /customer/cart/items` — `{ variantId, quantity }`; adds, or increments if the variant
  is already a line. `BUSINESS_RULE_FAILED` if no seller (preferred distributor, else admin)
  currently has stock for it, or the line's total quantity after this add would exceed what
  that seller has — same distributor-else-admin resolution `GET /cart`'s `offer` uses, so a
  line the customer can't checkout can't be added in the first place. `details: { variantId,
  available }` names the exact shortfall.
- `PATCH /customer/cart/items/:id` — `{ quantity }`; sets a line's quantity. 404 if the line
  isn't in the caller's cart; same `BUSINESS_RULE_FAILED` stock check as above (against the
  new quantity, not the delta) if it goes above what's available.
- `DELETE /customer/cart/items/:id` — removes one line (the "delete product [from cart]"
  action). 404 if the line isn't in the caller's cart.
- `DELETE /customer/cart` 🟡 — empties every line in one call (checkout-abandoned / "start
  over"), instead of making the client loop the line-delete above. No cart yet is not an
  error — it returns the same empty `CartView` `GET` would.

### `POST /auth/admin/login` 🟡
Who: ADMIN (via Google sign-in)        Permission: public (pre-auth)

Request
```ts
{ idToken: string }   // Firebase Google ID token obtained by the admin panel
```

Response `data`
```ts
{
  token: string,        // opaque session token — store client-side, send as Bearer
  expiresAt: string,    // ISO 8601
  account: {
    id: string, email: string | null, phone: string | null,
    name: string | null, avatarUrl: string | null,
    role: "ADMIN", status: "ACTIVE"
  }
}
```

Errors: `VALIDATION_FAILED` — no idToken · `UNAUTHENTICATED` — invalid/expired Firebase token ·
`FORBIDDEN` — email not in the admin allowlist, not an admin, or not active ·
`EXTERNAL_SERVICE_ERROR` — auth not configured on the server
Notes: creates the admin `Account` on first successful login. Server stores only the token hash.

### `GET  /auth/me` 🟡
Who: any authenticated
Response `data`: `{ account: { id, email, phone, name, avatarUrl, role, status } }`
Errors: `UNAUTHENTICATED`

### `POST /auth/logout` 🟡
Who: any authenticated
Response `data`: `{ ok: true }`
Notes: revokes the current session (`revokedAt`). Idempotent.

### `POST /auth/distributor/login` 🟡
Who: approved DISTRIBUTOR (via Firebase email/password or Google sign-in). Permission: public (pre-auth)
Request: `{ idToken: string }` — Firebase ID token from either sign-in method.
Response `data`: same shape as `POST /auth/admin/login` — `{ token, expiresAt, account }`.
The server requires a verified email on an `ACTIVE` distributor `Account` with a
`DistributorProfile`. It never creates an account at login.
Errors: `VALIDATION_FAILED` — missing token · `UNAUTHENTICATED` — invalid token or unverified
email · `FORBIDDEN` — email not linked to an active distributor · `EXTERNAL_SERVICE_ERROR` —
Firebase server verification unavailable.
Notes: distributor panel OTP endpoints `/auth/otp/request` and `/auth/otp/verify` are retired.
Customer and diary OTP endpoints remain separate and active.

### `POST /auth/distributor/password-reset/request` 🟡
Who: public (pre-auth)
Request: `{ email: string }` — normalized to lowercase.
Response `data`: `{ ok: true }` for eligible, unknown, inactive, and rate-limited addresses alike.
Notes: for approved active distributors, the server starts sending a branded Firebase password reset link from
`nexashopping9@gmail.com`. The request is rate limited by IP and email. The link is never
returned to the browser. The generic response does not confirm delivery. This endpoint requires server SMTP configuration.
If SMTP credentials are missing, it returns `EXTERNAL_SERVICE_ERROR` for every address.

### `POST /auth/refresh` ⚪       *(refresh-token rotation — added before Phase 5 payments)*

## Accounts  *(Phase 1)*

No schema change — `Account`/`AdminProfile`/`DistributorProfile` already exist from Phase 1's
identity migration. This module is the CRUD surface over them. Distributor **self**-onboarding
is not built; `POST /accounts/distributors` is the admin-controlled onboarding path. The admin
assigns an email; the server sends a welcome message with a password link. The distributor
can request another link from the panel's **Reset password** action.

```ts
type AccountSummary = {
  id: string, name: string | null, email: string | null, phone: string | null,
  avatarUrl: string | null, role: "ADMIN"|"COADMIN"|"DISTRIBUTOR"|"CUSTOMER",
  status: "PENDING"|"ACTIVE"|"SUSPENDED"|"CLOSED",
  createdAt: string, lastLoginAt: string | null,
  distributor: { businessName: string, referralCode: string, territory: string | null,
                 onboardedAt: string | null } | null,
}
```

### `GET /accounts` 🟡
Who: ADMIN     Permission: `account:read`

Query
```ts
{ role?: "ADMIN"|"COADMIN"|"DISTRIBUTOR"|"CUSTOMER", status?: "PENDING"|"ACTIVE"|"SUSPENDED"|"CLOSED",
  q?: string, cursor?: string, limit?: number }
```
Response `data`: `{ accounts: AccountSummary[] }`, `meta: { cursor, hasMore }`
Notes: `q` matches name/email/phone (`ILIKE` — this table is small; no tsvector needed, same
reasoning as inventory's search).

### `GET /accounts/admin` 🟡
Who: any authenticated
Response `data`: `{ account: AccountSummary }`
Errors: `NOT_FOUND` — no admin configured, or the configured admin has never logged in
Notes: who to buy central stock from. There's no real "platform seller" concept in the
schema (more than one `ADMIN` account is legal) — this resolves to the first entry in
`ADMIN_EMAILS` (the same allowlist that decides who *can* be an admin). A distributor's
"buy from admin" flow calls this once to get `sellerAccountId`, then uses it against
`GET /inventory?sellerAccountId=`, `POST /cart/items`, `POST /orders`.

### `GET /accounts/:id` 🟡
Who: ADMIN, or the account viewing itself
Response `data`: `{ account: AccountSummary }`
Errors: `NOT_FOUND` · `FORBIDDEN` — not admin and not your own account

### `PATCH /accounts/:id` 🟡
Who: ADMIN (any field below), or the account itself (`name`, `avatarUrl` only)
Notes: an admin cannot change their *own* `status` — there is no recovery path if it leaves
`ACTIVE` (the next request and even re-login both re-check status fresh), so a lone admin
locking themselves out would need direct DB access to undo. An admin can still change every
other admin's status, and every other field on their own account.

Request
```ts
{ name?: string, avatarUrl?: string, email?: string, // admin only, distributor accounts only
  status?: "PENDING"|"ACTIVE"|"SUSPENDED"|"CLOSED",     // admin only, not on their own account
  businessName?: string, territory?: string,            // admin only, distributor accounts only
}
```
Response `data`: `{ account: AccountSummary, welcomeEmailSent?: boolean }` — the email flag
appears when the admin changes an active distributor's login email or activates a distributor.
Errors: `VALIDATION_FAILED` · `NOT_FOUND` ·
`FORBIDDEN` — not admin and (not your own account, or trying to set an admin-only field)

### `POST /accounts/distributors` 🟡
Who: ADMIN     Permission: `account:write`
Notes: requires a login email and creates the `Account` and its `DistributorProfile` together,
in one transaction. The server also provisions a Firebase user with an undisclosed random
password if one does not already exist. The server sends a welcome email from the configured
Gmail sender containing a Firebase password link when the account is active. Pending
distributors receive the welcome link on activation. Admin email changes revoke existing
NexaShopping sessions and send a new welcome link when the account is active.
`referralCode` is generated server-side (unique, not client-supplied).

Request
```ts
{ name: string, email: string, phone: string, businessName: string, territory?: string,
  referredByAccountId?: string, status?: "PENDING" | "ACTIVE",   // default "ACTIVE"
  credit?: { enabled: boolean, limit: string } // decimal rupees; creates 30-day trade credit
}
```
Response `data`: `{ account: AccountSummary, welcomeEmailSent?: boolean }`. The email flag
appears for active accounts. The account is created even if mail delivery fails; Admin can
resend the link. Pending accounts receive their welcome link when activated.
Errors: `VALIDATION_FAILED` · `CONFLICT` — phone/email already in use or Firebase email is unverified ·
`NOT_FOUND` — referredByAccountId doesn't exist ·
`BUSINESS_RULE_FAILED` — referredByAccountId is not a distributor

### `POST /accounts/:id/welcome-email` 🟡
Who: ADMIN
Response `data`: `{ ok: true }`
Notes: generates a fresh Firebase password link and resends the welcome email to the
active distributor's current login email. If Gmail delivery fails, returns
`EXTERNAL_SERVICE_ERROR` so Admin can retry.

### Deliberately not built yet
**Co-admin creation** (roadmap Phase 1 item 4) needs a decision first: admin identity today is
driven entirely by the `ADMIN_EMAILS` env allowlist (see `nexashopping-auth` — an account is
auto-created on first Google login *if* the email is already allowlisted). Adding a
DB-driven "invite a co-admin" flow means deciding whether it replaces or layers on top of that
allowlist — not assumed here, `[DECISION NEEDED]`.

## Catalog  *(Phase 2)*

Every product has **at least one** variant — the request always includes a `variants` array
of length >=1 (a product with no axes still sends one "default" variant). See
`nexa-docs/docs/DATA-MODEL.md` §3.

### `GET /categories` 🟡
Who: any (public)     Permission: none

Query
```ts
{ parentId?: string, isActive?: "true" | "false" }
```
Response `data`: `{ categories: Category[] }` — flat list, no cursor (categories are a small tree).
```ts
type Category = {
  id: string, parentId: string | null, name: string, slug: string,
  imageUrl: string | null, position: number, isActive: boolean,
}
```

### `POST /categories` 🟡
Who: ADMIN     Permission: `category:write`

Request
```ts
{ name: string, slug?: string, parentId?: string, imageUrl?: string, position?: number }
```
Response `data`: `{ category: Category }`
Errors: `VALIDATION_FAILED` · `NOT_FOUND` — parentId doesn't exist · `CONFLICT` — slug taken
Notes: `slug` auto-generated from `name` when omitted.

### `PATCH /categories/:id` 🟡
Who: ADMIN     Permission: `category:write`

Request: any subset of the POST body, plus `{ isActive?: boolean }`
Response `data`: `{ category: Category }`
Errors: `VALIDATION_FAILED` · `NOT_FOUND` · `CONFLICT` — slug taken

### `GET /categories/:categoryId/attributes` 🟡
Who: any (public)
Response `data`: `{ attributes: AttributeDef[] }`
```ts
type AttributeDef = {
  id: string, categoryId: string, name: string, code: string,
  type: "TEXT" | "NUMBER" | "BOOLEAN" | "ENUM", unit: string | null,
  isVariantAxis: boolean, isFilterable: boolean, position: number,
  options: { id: string, value: string, position: number }[],
}
```

### `POST /categories/:categoryId/attributes` 🟡
Who: ADMIN     Permission: `category:write`
Notes: defines a variant axis (e.g. Size) or a filterable/descriptive attribute for the category.

Request
```ts
{
  name: string, code: string,                      // code: camelCase, stable — used by filters
  type?: "TEXT" | "NUMBER" | "BOOLEAN" | "ENUM",    // default "TEXT"
  unit?: string, isVariantAxis?: boolean, isFilterable?: boolean, position?: number,
  options?: string[],                               // seeds AttributeOption rows (ENUM axes)
}
```
Response `data`: `{ attribute: AttributeDef }`
Errors: `VALIDATION_FAILED` · `NOT_FOUND` — category · `CONFLICT` — code already used in this category

### `GET /products` 🟡
Who: any (public)     Permission: none

Query
```ts
{ categoryId?: string, status?: "DRAFT"|"ACTIVE"|"ARCHIVED", q?: string, cursor?: string, limit?: number }
```
Response `data`: `{ products: ProductSummary[] }`, `meta: { cursor, hasMore }`
```ts
type ProductSummary = {
  id: string, category: { id: string, name: string, slug: string },
  brand: string, name: string, slug: string, shortDescription: string,
  status: "DRAFT" | "ACTIVE" | "ARCHIVED", variantCount: number,
  minMrp: string | null, maxMrp: string | null,        // null only if a product has 0 active variants
  image: { url: string, alt: string | null } | null,
}
```
Notes: `q` runs a ranked full-text search (Postgres tsvector) and returns a single page —
`meta.hasMore` is always `false` and `cursor` always `null` when `q` is set. Without `q`,
results are cursor-paginated, newest first.

### `POST /products` 🟡
Who: ADMIN     Permission: `product:write`

Request
```ts
{
  categoryId: string, brand: string, name: string, slug?: string,
  shortDescription: string, description?: string,
  status?: "DRAFT" | "ACTIVE" | "ARCHIVED",             // default "DRAFT"
  variants: {
    sku: string, name: string, mrp: string,             // mrp: decimal string, e.g. "499.00"
    options?: { attributeDefId: string, value: string }[],
  }[],                                                   // min length 1
  media?: { url: string, alt?: string, isPrimary?: boolean }[],
}
```
Response `data`: `{ product: ProductDetail }` (see GET /products/:id for the shape)
Errors: `VALIDATION_FAILED` · `NOT_FOUND` — categoryId ·
`CONFLICT` — slug or sku already in use · `BUSINESS_RULE_FAILED` — two variants share the
same combination of attribute values

### `GET /products/:id` 🟡
Who: any (public)
Response `data`: `{ product: ProductDetail }`
```ts
type ProductDetail = {
  id: string, category: { id: string, name: string, slug: string },
  brand: string, name: string, slug: string, shortDescription: string,
  description: string | null, status: "DRAFT"|"ACTIVE"|"ARCHIVED", archivedAt: string | null,
  variants: {
    id: string, sku: string, name: string, mrp: string, isActive: boolean,
    options: { attributeDefId: string, name: string, code: string, value: string }[],
    media: { id: string, url: string, alt: string | null, isPrimary: boolean, position: number }[],
  }[],
  media: { id: string, url: string, alt: string | null, isPrimary: boolean, position: number }[],
  attributes: { attributeDefId: string, name: string, code: string, value: string }[],
  createdAt: string, updatedAt: string,
}
```
Errors: `NOT_FOUND`

### `PATCH /products/:id` 🟡
Who: ADMIN     Permission: `product:write`

Request: any subset of `{ categoryId, brand, name, slug, shortDescription, description, status }`
Response `data`: `{ product: ProductDetail }`
Errors: `VALIDATION_FAILED` · `NOT_FOUND` · `CONFLICT` — slug taken
Notes: setting `status: "ARCHIVED"` stamps `archivedAt`; setting any other status clears it.

### `POST /products/:id/variants` 🟡
Who: ADMIN     Permission: `product:write`

Request
```ts
{ sku: string, name: string, mrp: string, options?: { attributeDefId: string, value: string }[] }
```
Response `data`: `{ variant: ProductDetail["variants"][number] }`
Errors: `VALIDATION_FAILED` · `NOT_FOUND` — product · `CONFLICT` — sku taken ·
`BUSINESS_RULE_FAILED` — this combination of attribute values already exists on the product

### Deferred to a later pass
- **Media upload** — `media[].url` is a plain string today (the caller already has a
  hosted URL). Signed direct-to-storage upload URLs (ROADMAP.md Phase 2 item 4) are not
  built yet; add `POST /uploads/sign` when storage is wired up.

## Inventory  *(Phase 3)*

One `StockItem` per (owner, variant) — admin is just the owner whose role is `ADMIN`, no
special case. `GET /inventory` is scoped to the caller automatically, the same way `GET
/orders` will be later — no `ownerAccountId` query param exists because there's nothing to
pass, you only ever see your own stock through this API. See `nexa-docs/docs/DATA-MODEL.md`
§4. `onHand`/`reserved` are only ever written by `inventory.service.applyDelta()`, alongside a
`StockLedger` row, inside a transaction with the row locked — never by a plain `UPDATE`.

```ts
type StockItemView = {
  id: string,
  variant: { id: string, sku: string, name: string, mrp: string,
             product: { id: string, name: string, slug: string, brand: string } },
  onHand: number, reserved: number, available: number,   // available = onHand - reserved
  sellPrice: string, discountPrice: string | null,
  minimumRetailPrice: string | null, customerPrice: string | null,
  taxRatePct: string,
  isListed: boolean, lowStockAt: number | null,
}
```

### `GET /inventory` 🟡
Who: ADMIN · DISTRIBUTOR     Permission: none beyond being that role

Query
```ts
{ sellerAccountId?: string, isListed?: "true"|"false", lowStock?: "true", q?: string, cursor?: string, limit?: number }
```
Response `data`: `{ items: StockItemView[] }`, `meta: { cursor, hasMore }`
Notes: two modes. **Omit `sellerAccountId`** (or pass your own id) for "my stock" — every
field, every listing state, exactly as before. **Pass someone else's `sellerAccountId`** to
browse *their* storefront instead — this forces `isListed: true` and ignores `lowStock`
regardless of what you asked for, because a buyer has no business seeing another account's
private stock state. This is how a distributor shops admin's catalog: `GET /accounts/admin`
for the id, then `GET /inventory?sellerAccountId=<that id>`.
`lowStock=true` (own-stock mode only) returns only items where `available <= lowStockAt` (and
`lowStockAt` is set). `q` matches variant SKU or product name (plain `ILIKE` — this list is bounded by
what one owner stocks, not the whole catalog, so no tsvector needed here).

### `GET /inventory/export` 🟡
Who: ADMIN · DISTRIBUTOR — own stock only
Query: `{ format?: "csv" | "pdf" }` (default `"csv"`)
Response: `text/csv` or `application/pdf` (not the standard JSON envelope), `Content-Disposition: attachment`
Notes: point-in-time snapshot, no date range (this is "right now," not a period). Columns: SKU,
Product Name, Variant, Brand, On Hand, Reserved, Available, Listed, Sell Price, Discount Price,
Low Stock At. Registered ahead of `/:id` so Express never mistakes `export` for a stock-item id.
The PDF carries the same company letterhead (legal name, GSTIN, address, contact) as the GST
invoice PDF (`invoices/invoice-pdf.ts`) — see `core/report-pdf.ts`, the shared builder every
export PDF in this API uses.

### `GET /inventory/ledger/export` 🟡
Who: ADMIN · DISTRIBUTOR — own stock only
Query: `{ from: "YYYY-MM-DD", to: "YYYY-MM-DD", format?: "csv" | "pdf" }` (default `"csv"`)
Response: `text/csv` or `application/pdf`, `Content-Disposition: attachment`
Notes: owner-wide movement history across **every** stock item the caller holds, not one item
at a time (unlike `GET /inventory/:id/ledger`) — `StockLedger` has no `ownerAccountId` column,
so this joins through the owning `StockItem`. Ordered oldest-first. Columns: Date, SKU, Product
Name, Reason, Delta, On Hand After, Reserved After, Ref Type, Note. (No internal record IDs —
exports never include raw UUIDs.)

### `GET /inventory/ledger/recent` 🟡
Who: ADMIN · DISTRIBUTOR — own stock only
Query: `{ limit?: number }` (1–20, default 5)
Response `data`: `{ entries: Array<{ id, createdAt, sku, productName, reason, delta, onHandAfter, reservedAfter }> }`
Notes: same owner-wide join as the ledger export, but newest-first with no date range — built
for a small "recent activity" widget (the distributor dashboard's Inventory page), not for
browsing the full ledger.

### `POST /inventory` 🟡
Who: ADMIN · DISTRIBUTOR     Permission: none beyond being that role
Notes: creates the caller's `StockItem` for a variant if it doesn't exist yet (starts at
`onHand: 0`, unlisted), then receives stock into it — the one way stock enters the platform
from outside (a real supplier for admin; this is *not* how a distributor gets stock, that's
`POST /inventory/transfer`, but nothing stops a distributor recording their own direct
purchases the same way).

Request
```ts
{
  variantId: string, quantity: number,          // quantity > 0
  sellPrice: string, discountPrice?: string, taxRatePct?: string,
  unitCost?: string, batchNo?: string, expiryAt?: string, note?: string,
}
```
Response `data`: `{ item: StockItemView }`
Errors: `VALIDATION_FAILED` · `NOT_FOUND` — variant
Notes: when ADMIN creates a stock item for the first time, `minimumRetailPrice` and
`customerPrice` are required and must satisfy `sellPrice <= minimumRetailPrice <=
customerPrice <= MRP`. A distributor's first-stock price is checked against the primary
admin's configured retail range.

### `PATCH /inventory/:id` 🟡
Who: the item's owner     Permission: none beyond ownership
Notes: pricing/listing only — never touches `onHand`/`reserved`. Use `POST /inventory/:id/
adjust` for quantity corrections.

Request
```ts
{ sellPrice?: string, discountPrice?: string | null,
  minimumRetailPrice?: string, customerPrice?: string,
  taxRatePct?: string, isListed?: boolean, lowStockAt?: number }
```
Response `data`: `{ item: StockItemView }`
Errors: `VALIDATION_FAILED` · `NOT_FOUND` · `FORBIDDEN` — not the owner or distributor trying
to set admin retail boundaries · `BUSINESS_RULE_FAILED` — retail ordering is invalid. Admin
stock enforces wholesale <= minimum retail <= customer price <= MRP. Distributor effective
price must remain between the primary admin's minimum retail and customer prices.

### `GET /inventory/:id/retail-range` 🟡
Who: ADMIN or DISTRIBUTOR     Permission: stock-item owner only

Response `data`:
```ts
{ range: { minimumRetailPrice: string, maximumRetailPrice: string, mrp: string } }
```
For distributor stock, the minimum and maximum come from the primary admin's current
pricing for the same variant. This lets the distributor panel display the permitted
customer-price range before an update is submitted. The server still enforces the range.
Errors: `NOT_FOUND` · `FORBIDDEN` · `BUSINESS_RULE_FAILED` — admin range is not configured

### `POST /inventory/:id/adjust` 🟡
Who: the item's owner
Request
```ts
{ delta: number, reason: "ADJUSTMENT" | "DAMAGE" | "RETURN_IN", note?: string }
```
Response `data`: `{ item: StockItemView }`
Errors: `VALIDATION_FAILED` · `NOT_FOUND` · `FORBIDDEN` — not the owner ·
`BUSINESS_RULE_FAILED` — would take onHand negative
Notes: `note` required when `reason: "ADJUSTMENT"` (an unexplained correction is a bug
waiting to happen).

### `POST /inventory/transfer` 🟡
Who: ADMIN only (Phase 3 scope — admin to distributor; peer transfers are not built)
Notes: one transaction — `TRANSFER_OUT` on the admin's `StockItem`, `TRANSFER_IN` on the
distributor's (created if it doesn't exist yet, priced at `sellPrice` or the variant's `mrp`
if omitted).

Request
```ts
{ variantId: string, toAccountId: string, quantity: number, unitCost?: string, sellPrice?: string, note?: string }
```
Response `data`: `{ fromItem: StockItemView, toItem: StockItemView }`
Errors: `VALIDATION_FAILED` · `NOT_FOUND` — variant/toAccountId ·
`FORBIDDEN` — toAccountId is not an active distributor ·
`BUSINESS_RULE_FAILED` — insufficient stock on the admin's item

### `GET /inventory/:id/ledger` 🟡
Who: the item's owner
Query: `{ cursor?: string, limit?: number }`
Response `data`
```ts
{
  entries: {
    id: string, delta: number, onHandAfter: number, reservedAfter: number,
    reason: string, refType: string | null, refId: string | null,
    unitCost: string | null, batchNo: string | null, note: string | null,
    createdAt: string,
  }[]
}
```
Errors: `NOT_FOUND` · `FORBIDDEN` — not the owner

## Distributor Customers

```ts
type CustomerRelationshipView = {
  id: string,
  customer: { id: string, name: string | null, phone: string, status: AccountStatus },
  displayName: string | null, notes: string | null, status: "ACTIVE" | "BLOCKED",
  acquisitionSource: "ORGANIC" | "REFERRAL" | "DISTRIBUTOR_ASSISTED" | null,
  acquiredByAccountId: string | null, acquiredAt: string | null,
  createdAt: string, updatedAt: string,
}
```

### `POST /customers`
Who: active DISTRIBUTOR only

Request
```ts
{ phone: string, name: string, displayName?: string, notes?: string }
```
Response `data`: `{ customer: CustomerRelationshipView }` (`201` for a new relationship,
`200` when the same distributor repeats the attachment).
Errors: `VALIDATION_FAILED` · `FORBIDDEN` · `CONFLICT` — the phone belongs to a non-customer
account. The conflict response never reveals the existing account role or identity.
Notes: phone is E.164 and globally unique. A new account receives immutable
`DISTRIBUTOR_ASSISTED` acquisition; an existing customer's acquisition is never overwritten.

### `GET /customers`
Who: active DISTRIBUTOR only
Query: `{ q?: string, status?: "ACTIVE"|"BLOCKED", cursor?: string, limit?: number }`
Response `data`: `{ customers: CustomerRelationshipView[] }`, `meta: { cursor, hasMore }`
Notes: always scoped to the authenticated distributor.

### `GET /customers/:id`
Who: the relationship's distributor
Response `data`: `{ customer: CustomerRelationshipView }`
Errors: `NOT_FOUND` for missing and other-distributor relationships alike.

### `PATCH /customers/:id`
Who: the relationship's distributor
Request: `{ displayName?: string|null, notes?: string|null, status?: "ACTIVE"|"BLOCKED" }`
Response `data`: `{ customer: CustomerRelationshipView }`
Errors: `VALIDATION_FAILED` · `NOT_FOUND`

## Cart & Orders  *(Phase 4)*

**Status note:** `DATA-MODEL.md` marks Orders as a design sketch pending Q9 (invoicing/tax —
GST breakdown, invoice numbering, seller-of-record). Built here with the flat-tax model
already used by `StockItem.taxRatePct` (no CGST/SGST split) and **seller-of-record resolved as
"whoever's `StockItem` the order draws from"** — admin for their own stock, a distributor for
theirs, consistent with how ownership already works in Inventory (Phase 3). A GST-compliant
invoice layer is not built and would need Q9 answered properly first.

A cart always belongs to one buyer **and one seller** — `sellerAccountId` says whose
`StockItem` rows it prices against (a customer's distributor, or a distributor buying from
admin). There is no "default seller"; the caller always states it. Carts re-price **live** on
every read from `core/pricing.ts` — nothing price-related is ever stored on a `CartItem`.

```ts
type CartLine = {
  id: string, variantId: string, sku: string, name: string, quantity: number,
  unitPrice: string, discountAmount: string, taxAmount: string, lineTotal: string,
  available: number,   // the seller's current available stock for this variant
}
type CartView = {
  id: string | null,   // null until the first item is added — GET never creates a row
  sellerAccountId: string, status: "ACTIVE",
  items: CartLine[],
  subtotal: string, discountTotal: string, taxTotal: string, shippingTotal: string, grandTotal: string,
}
```

### `GET /cart` 🟡
Who: any authenticated (the buyer)
Query: `{ sellerAccountId: string, buyerAccountId?: string }`
Response `data`: `{ cart: CartView }`
Errors: `VALIDATION_FAILED` · `NOT_FOUND` — sellerAccountId doesn't exist ·
`FORBIDDEN` — buyerAccountId given but the caller is neither that buyer nor the seller
Notes: if the buyer has no active cart with this seller yet, returns an empty `CartView`
(`id: null`, `items: []`) rather than 404 — no row is created until something is added.
`buyerAccountId` defaults to the caller; only the seller (or admin) may pass someone else's —
this is how a distributor views a cart they're building for a walk-in customer
(DISTRIBUTOR_ASSISTED, same pattern as `POST /orders`).

### `POST /cart/items` 🟡
Who: any authenticated
Notes: **adds** to the line's quantity (creates the line, and the cart, if either doesn't
exist yet). Use `PATCH` to set an absolute quantity.

Request
```ts
{ sellerAccountId: string, buyerAccountId?: string, variantId: string, quantity: number }   // quantity > 0
```
Response `data`: `{ cart: CartView }`
Errors: `VALIDATION_FAILED` · `NOT_FOUND` — sellerAccountId or variantId ·
`BUSINESS_RULE_FAILED` — seller does not list this variant ·
`FORBIDDEN` — buyerAccountId given but the caller is neither that buyer nor the seller
Notes: `buyerAccountId` same rule as `GET /cart` — omit it to add to your own cart; a seller
passes it to build a cart for a customer they're assisting.

### `PATCH /cart/items/:id` 🟡
Who: the cart's buyer, or the cart's seller (assisting)
Request: `{ quantity: number }` (>= 1 — use DELETE to remove a line)
Response `data`: `{ cart: CartView }`
Errors: `VALIDATION_FAILED` · `NOT_FOUND` · `FORBIDDEN` — not your cart

### `DELETE /cart/items/:id` 🟡
Who: the cart's buyer, or the cart's seller (assisting)
Response `data`: `{ cart: CartView }`
Errors: `NOT_FOUND` · `FORBIDDEN` — not your cart

---

```ts
type OrderStatus = "AWAITING_PAYMENT" | "CONFIRMED" | "SHIPPED" | "DELIVERED" | "CANCELLED"
type Order = {
  id: string, orderNo: string, buyerAccountId: string, sellerAccountId: string,
  channel: "WEB" | "APP" | "DISTRIBUTOR_ASSISTED",
  status: OrderStatus, paymentStatus: "UNPAID" | "PENDING" | "PAID",
  paymentMethod: "PHONEPE" | "COD" | "CREDIT" | null,
  fulfilmentStatus: "PENDING" | "SHIPPED" | "DELIVERED",
  subtotal: string, discountTotal: string, taxTotal: string, shippingTotal: string, grandTotal: string,
  currency: "INR",
  shippingAddress: object, billingAddress: object,   // snapshots, not FKs
  items: { id: string, sku: string, productName: string, variantLabel: string,
            unitPrice: string, quantity: number, discount: string, taxRatePct: string,
            taxAmount: string, lineTotal: string }[],
  placedAt: string, confirmedAt: string | null, shippedAt: string | null,
  deliveredAt: string | null, cancelledAt: string | null, cancelReason: string | null,
}
```
For PhonePe orders, only a backend-verified provider success moves `paymentStatus` to `PAID`
and the order to `CONFIRMED`. Pending or uncertain payments keep stock reserved. A definitive
provider failure or expiry cancels the order and releases its reservation exactly once.

### `POST /orders` 🟡
Who: any authenticated (buyer channel), or a distributor placing FOR a customer
(`channel: "DISTRIBUTOR_ASSISTED"`)
Notes: one transaction — reserve stock (per `inventory.service.reserveStock`, RESERVE ledger
entries) → price via `core/pricing.ts` → create `Order` + `OrderItem`s (address snapshotted as
JSONB) → clear the source cart (`status: CONVERTED`). All or nothing (ARCHITECTURE.md
§6 Transactions).

Request
```ts
{
  sellerAccountId: string,
  channel: "WEB" | "APP" | "DISTRIBUTOR_ASSISTED",
  paymentMethod?: "PHONEPE" | "CREDIT", // active distributor buying central admin stock
  buyerAccountId?: string,        // required + only meaningful when channel is DISTRIBUTOR_ASSISTED
  shippingAddress: { contactName: string, contactPhone: string, line1: string, line2?: string,
                      city: string, state: string, pincode: string, country?: string },
  billingAddress?: object,        // defaults to shippingAddress if omitted
}
```
Response `data`: `{ order: Order, payment?: { merchantOrderId: string, status: "PENDING"|"SUCCESS", redirectUrl: string } }`
Errors: `VALIDATION_FAILED` · `NOT_FOUND` — seller/buyer · `BUSINESS_RULE_FAILED` — cart is
empty, or insufficient stock on one or more lines (nothing is reserved if any line fails)
Notes: prices the buyer's **current cart** with this seller — there is no separate "line
items in the request body"; add to the cart first, then place the order from it.
For distributor central-stock purchases, `paymentMethod: "PHONEPE"` creates the order and
payment intent atomically, then initiates PhonePe after the database transaction commits.
Open the returned HTTPS `redirectUrl` in the browser. PhonePe returns to the configured
distributor payment-response page with `merchantOrderId`; the browser must verify through the
backend status endpoint and must never mark the order paid locally.
`paymentMethod: "CREDIT"` confirms the order immediately and atomically posts a 30-calendar-day
credit charge. Suspended accounts, overdue charges, or an insufficient available limit reject
the complete checkout without reserving stock or converting the cart.

### `GET /orders` 🟡
Who: any authenticated — scoped automatically: buyers see their own orders, sellers see their
sales (the same endpoint for everyone, per CONVENTIONS.md — scoping happens in the service)
Query: `{ role?: "buyer"|"seller", status?: OrderStatus, cursor?: string, limit?: number }`
Response `data`: `{ orders: Order[] }`, `meta: { cursor, hasMore }`
Notes: `role` disambiguates when an account is both (e.g. a distributor is a seller to
customers and a buyer from admin); defaults to showing both sides combined.

### `GET /orders/export` 🟡
Who: an active `DISTRIBUTOR` or `ADMIN` — their own sales only, never another account's
Query: `{ from: "YYYY-MM-DD", to: "YYYY-MM-DD", format?: "csv" | "pdf" }` (default `"csv"`)
Response: `text/csv` or `application/pdf` (not the standard JSON envelope), `Content-Disposition: attachment`
Notes: GST sales register — one row per **invoiced** order (an order with no `Invoice` is
excluded entirely, so an un-shipped order never appears). Filtered by the invoice's own
`issuedAt` — the accounting date — not `placedAt`. Columns: Invoice Date, Invoice No, Order No,
Customer Name, Customer Phone, Channel, Payment Method, Payment Status, Tax Rate %, Taxable
Value (subtotal − discount), Tax Amount, Grand Total. `Tax Rate %` reads `"Mixed"` when an
order's line items don't share one flat rate. Registered as `/orders/export`, ahead of `/:id`,
so Express never mistakes `export` for an order ID.

### `GET /orders/:id` 🟡
Who: the buyer, the seller, or admin
Response `data`: `{ order: Order }`
Errors: `NOT_FOUND` · `FORBIDDEN`

### `GET /orders/:id/payment-status` live
Who: the order buyer (DISTRIBUTOR), the seller, or admin
Response `data`:
```ts
{ payment: {
  merchantOrderId: string,
  providerStatus: "CREATED" | "PENDING" | "SUCCESS" | "FAILED" | "EXPIRED",
  paymentStatus: "UNPAID" | "PENDING" | "PAID",
  orderStatus: OrderStatus,
  redirectUrl: string | null,
  verifiedAt: string | null,
} }
```
Errors: `NOT_FOUND` · `FORBIDDEN` · `CONFLICT` — order does not use PhonePe ·
`EXTERNAL_SERVICE_ERROR` — PhonePe status is temporarily unavailable
Notes: queries PhonePe only while the intent is non-terminal, then applies the observation
through the idempotent backend finalizer.

### `GET /orders/:id/invoice` 🟡
Who: the buyer, the seller, or admin — same visibility as `GET /orders/:id`
Response `data`: `{ invoice: { invoiceNo: string, pdfUrl: string, issuedAt: string } }`
Errors: `NOT_FOUND` — order not found, actor not permitted, or (see Notes) not yet generated
Notes: NexaShopping is seller of record on the invoice regardless of which StockItem fulfilled
the order (ADR-0016) — the PDF never names the fulfilling distributor as seller. `invoiceNo` is
a Postgres-sequence-backed number (`NX-INV-000123`), same pattern as `Order.orderNo`. Generated
once, automatically, when the order transitions to `SHIPPED` — not `DELIVERED` — per CGST Act
Section 31 / Rule 46 (the tax invoice must issue at or before removal of goods for supply, i.e.
dispatch, not whenever delivery later gets confirmed). An order not yet shipped has no invoice
and this returns `NOT_FOUND` until then. `pdfUrl` is a Cloudinary URL (`resource_type: raw`);
the client links/opens it directly, no separate download endpoint.

### `POST /orders/:id/cancel` 🟡
Who: the buyer (before `SHIPPED`), the seller, or admin
Notes: one transaction — releases the reserved stock (RELEASE ledger entries) and writes an
`OrderEvent`. Legal only from `AWAITING_PAYMENT` or `CONFIRMED` (the order-status state
machine lives in `orders.service`, not the client).

Request: `{ reason?: string }`
Response `data`: `{ order: Order }`
Errors: `NOT_FOUND` · `FORBIDDEN` · `CONFLICT` — order is already shipped/delivered/cancelled,
or is a PhonePe order (refunds are not enabled)

### `POST /orders/:id/confirm` 🟡
Who: the seller, or admin
Notes: `AWAITING_PAYMENT → CONFIRMED`. PhonePe orders cannot be manually confirmed before
backend payment verification. Other legacy/manual orders stamp `confirmedAt` and write an `OrderEvent`.
Response `data`: `{ order: Order }`
Errors: `NOT_FOUND` · `FORBIDDEN` — not the seller/admin ·
`BUSINESS_RULE_FAILED` — order isn't `AWAITING_PAYMENT`

### `POST /orders/:id/ship` 🟡
Who: the seller, or admin
Notes: `CONFIRMED → SHIPPED`. Still no stock movement — the reservation holds until delivery.
Stamps `shippedAt`, sets `fulfilmentStatus: "SHIPPED"`, writes an `OrderEvent`.
Response `data`: `{ order: Order }`
Errors: `NOT_FOUND` · `FORBIDDEN` · `BUSINESS_RULE_FAILED` — order isn't `CONFIRMED`

### `POST /orders/:id/deliver` 🟡
Who: the seller, or admin
Notes: `SHIPPED → DELIVERED` — the transition that actually moves stock. One transaction, per
line: the seller's reservation is consumed (`RELEASE` then `SALE_OUT` ledger entries — the
stock has genuinely left them now), and **if the buyer is a `DISTRIBUTOR`**, the same quantity
lands in the buyer's own `StockItem` (`TRANSFER_IN`, created if it doesn't exist yet, priced at
the order's `unitPrice` — adjustable afterward via `PATCH /inventory/:id`). A `CUSTOMER` buyer
gets no `StockItem` — they don't resell. This is what closes the loop between "place an order"
and a distributor's own inventory actually reflecting it; before this endpoint, the only way
stock reached a distributor was the separate `POST /inventory/transfer`, which still exists
and still works independently.
Response `data`: `{ order: Order }`
Errors: `NOT_FOUND` · `FORBIDDEN` · `BUSINESS_RULE_FAILED` — order isn't `SHIPPED`

### Deliberately not built yet
- **Order expiry** (unpaid orders auto-cancelling) exists as `nexa-server/scripts/
  expire-orders.ts` but is **not scheduled anywhere**. It only touches orders still stuck at
  `AWAITING_PAYMENT` (a confirmed/shipped/delivered order is untouched), but there's still no
  real payment gateway (Phase 5) to define what "timed out" actually means — pick a threshold
  with the user before wiring it into crontab. Also: not a BullMQ worker, a plain script —
  Redis is deferred to before Phase 5 per `nexa-server/CLAUDE.md` (Q7 unresolved).
- **GST-compliant invoicing** (HSN codes, CGST/SGST/IGST split, per-seller invoice numbering)
  — Q9, unanswered. What's built is enough to place, price, and fulfil an order correctly; it
  is not enough to hand a customer a legally compliant tax invoice.

## Payments  *(Phase 5)*

### `POST /payments/phonepe/callback` live
Who: PhonePe (SDK-validated callback credentials and raw body)
Response `data`: `{ accepted: true, duplicate: boolean }`
Notes: provider payloads are sanitized before storage. Callback and status-query races lock
the same `PaymentIntent` and produce one terminal business effect.

Refund endpoints are intentionally absent. Paid PhonePe orders cannot be cancelled until
refund capability is activated and a separately approved refund policy is implemented.

## Credit  *(Phase 6)*

All money fields are decimal strings. A distributor may read only its own account; ADMIN may
read or update any distributor credit account. Available credit is always
`creditLimit - currentBalance`. Eligibility is evaluated at request time, without cron.

### `GET /credit/:accountId` 🟡
Response `data`: `{ credit: { accountId, distributorAccountId, creditLimit, currentBalance,
availableCredit, status, termDays, hasOverdueCharges, nextDueAt } }`

### `GET /credit/:accountId/charges` 🟡
Response `data`: `{ charges: CreditCharge[] }` ordered oldest due first.

### `GET /credit/:accountId/ledger` 🟡
Response `data`: `{ entries: CreditLedgerEntry[] }` ordered newest first.

### `GET /credit/:accountId/ledger/export` 🟡
Who: the distributor that owns the credit account, or admin — same as the other credit reads
Query: `{ from: "YYYY-MM-DD", to: "YYYY-MM-DD", format?: "csv" | "pdf" }` (default `"csv"`)
Response: `text/csv` or `application/pdf` (not the standard JSON envelope), `Content-Disposition: attachment`
Notes: a statement export, ordered **oldest first** (unlike the UI's newest-first list) so it
reads top-to-bottom as a running balance, the same way a bank statement does. Columns: Date,
Direction, Reason, Amount, Balance After, Source Type, Note. (No internal record IDs — exports
never include raw UUIDs.)

### `GET /credit/:accountId/repayments` 🟡
Response `data`: `{ repayments: CreditRepayment[] }` ordered newest first.

### `POST /credit/:accountId/repayments/phonepe` 🟡
Who: the distributor that owns the credit account
Request: `{ amount: string, idempotencyKey: string }`
Response `data`: `{ repayment, payment: { merchantOrderId, status, redirectUrl } }`
Notes: amount must be greater than zero and no greater than the outstanding balance. A suspended
or overdue account may repay. Only one PhonePe repayment may remain pending at a time.

### `GET /credit/:accountId/repayments/:repaymentId/payment-status` 🟡
Who: the distributor that owns the credit account, or ADMIN
Response `data`: `{ payment: { merchantOrderId, providerStatus, repaymentStatus,
redirectUrl, verifiedAt } }`
Notes: the backend queries PhonePe while pending and applies a verified success through the
same idempotent oldest-due-first allocator used by callbacks.

### `POST /credit/:accountId/repayments/offline` 🟡
Who: ADMIN
Request: `{ amount: string, paidAt: string, methodLabel: string, externalReference: string }`
Notes: the external reference is mandatory and unique per credit account. The repayment,
oldest-due-first allocations, balance reduction, and ledger entry are one transaction.

### `PATCH /credit/:accountId` 🟡
Who: ADMIN
Request: `{ creditLimit?: string, status?: "ACTIVE" | "SUSPENDED", reason: string }`
Notes: a new limit may not be lower than the current outstanding balance. Every limit/status
change is auditable; no direct balance adjustment is accepted through this endpoint.

Credit orders can be cancelled only before dispatch and only while the complete original
charge remains unpaid. Cancellation releases stock, reverses the charge, writes a credit
ledger entry, and reduces the balance in the same transaction. A partially repaid or shipped
credit order cannot be cancelled.

## Settlements  *(Phase 6)*

Distributor-fulfilled customer PhonePe orders create one full-value `HELD` payable after
verified payment. Delivery changes it to `PAYABLE`. COD, admin-fulfilled customer orders, and
distributor inventory purchases never create a distributor payable.

### `GET /settlements` 🟡
Who: ADMIN
Query: `{ status?: "HELD" | "PAYABLE" | "PAID", distributorAccountId?: string }`
Response `data`: `{ payables: DistributorPayable[] }`

### `GET /settlements/export` 🟡
Who: ADMIN · DISTRIBUTOR — own payables only
Query: `{ from: "YYYY-MM-DD", to: "YYYY-MM-DD", format?: "csv" | "pdf" }` (default `"csv"`)
Response: `text/csv` or `application/pdf` (not the standard JSON envelope), `Content-Disposition: attachment`
Notes: date range filters on the payable's `createdAt`, ordered oldest-first. This is the
distributor's proof of what NexaShopping actually paid them — the record to check against if a
payout is ever disputed. Columns: Order No, Amount, Status, Eligible Date, Paid Date, Payout
Reference, Method. (No internal record IDs — exports never include raw UUIDs.) Registered ahead
of any future parameterized route so Express never mistakes `export` for an id.

### `GET /settlements/payouts` 🟡
Who: ADMIN
Query: `{ distributorAccountId?: string }`
Response `data`: `{ payouts: DistributorPayout[] }`

### `POST /settlements/payouts` 🟡
Who: ADMIN
Request: `{ payableIds: string[], method: string, paidAt: string, externalReference: string }`
Notes: every selected payable must be `PAYABLE` and belong to one distributor. The payout,
allocations, and payable `PAID` transitions are atomic. Reusing the same distributor/reference
with the identical payable set returns the original payout; conflicting reuse is rejected.


---

## Diary cash-receipt ledger (October 2026)

Base path: `/diary`. All routes require an existing authenticated, active account. `ADMIN` alone may use `/admin/*`; `COADMIN` is denied. Distributor submissions are limited to active customers linked to that distributor. Customer and distributor self-service reads return approved entries only.

This is a record of actual cash handovers, not a payment gateway or installment scheduler. One handover creates one entry, even if the amount covers multiple months. No separate diary enrollment record exists. Amounts are INR decimal strings; timestamps are ISO 8601 with an offset on input and UTC in responses. Approved, non-reversed entries alone count toward totals.

### Read routes

| Method and path | Role | Response `data` |
| --- | --- | --- |
| `GET /diary/me/summary` | CUSTOMER, DISTRIBUTOR | `{ account, approvedTotal, currency: "INR", approvedEntryCount, plan }`; `plan` is static reference metadata (`monthlyReferenceAmount: "1000.00"`, `durationMonths: 72`), not a schedule |
| `GET /diary/me/entries?page=&limit=` | CUSTOMER, DISTRIBUTOR | `{ entries: DiaryEntry[] }`; approved entries only, page metadata |
| `GET /diary/distributor/customers?page=&limit=` | DISTRIBUTOR | `{ customers: DiaryPerson[] }`; active linked customers, page metadata |
| `GET /diary/distributor/entries?page=&limit=&status=` | DISTRIBUTOR | `{ entries: DiaryEntry[] }`; own submissions, page metadata |
| `GET /diary/admin/overview` | ADMIN | `{ approvedTotal, currency: "INR", approvedEntryCount, pendingCount, recentEntries }` |
| `GET /diary/admin/customers?query=&page=&limit=` | ADMIN | `{ customers: DiaryPerson[] }`; active customer search, page metadata |
| `GET /diary/admin/customers/:id?page=&limit=` | ADMIN | `{ customer, approvedTotal, entries: DiaryEntry[] }`; page metadata |
| `GET /diary/admin/entries?page=&limit=&status=&customerAccountId=&distributorAccountId=&from=&to=` | ADMIN | `{ entries: DiaryEntry[] }`; page metadata |

`status` may be `PENDING`, `APPROVED`, `REJECTED`, or `REVERSED`. Admin date filters are ISO timestamps with offsets. `page` defaults to 1 and `limit` to 20, with a maximum of 100.

### Write routes

| Method and path | Role | Request | Result |
| --- | --- | --- | --- |
| `POST /diary/distributor/entries` | DISTRIBUTOR | `{ customerAccountId, amount, cashReceivedAt, notes?, idempotencyKey }` | Creates `PENDING` entry; approval is separate |
| `POST /diary/admin/entries` | ADMIN | Same fields plus `confirmed: true` | Creates an `APPROVED` entry after explicit confirmation |
| `POST /diary/admin/entries/:id/approve` | ADMIN | `{}` | `PENDING` to `APPROVED` |
| `POST /diary/admin/entries/:id/reject` | ADMIN | `{ reason }` | `PENDING` to `REJECTED` |
| `POST /diary/admin/entries/:id/reverse` | ADMIN | `{ reason }` | `APPROVED` to `REVERSED`, retaining audit history |

Creation returns `{ entry: DiaryEntry }`, HTTP 201 for a new entry or HTTP 200 for an identical idempotent replay. Review routes return `{ entry: DiaryEntry }`. `amount` must be a positive decimal string with at most two decimal places; `notes` is at most 500 characters; `idempotencyKey` is 8-128 characters; a rejection or reversal `reason` is 3-500 characters. The service also rejects implausible cash receipt dates and inactive or ineligible accounts.

`DiaryEntry` includes `id`, `referenceCode`, `customer`, `customerAccountId`, `collectorDistributorAccountId`, `collectorDistributor`, `createdByAccountId`, `amount`, `currency`, `cashReceivedAt`, `createdAt`, `status`, `notes`, `reason`, `approvedAt`, `rejectedAt`, and `auditEvents`. Audit events retain actor, transition, timestamp, and limited metadata. Approved amounts are not edited in place; corrections use a reasoned reversal.

---

## Changelog

Record every breaking change here, so a panel that has fallen behind knows what to fix.

| Date | Endpoint | Change | Breaking? |
|---|---|---|---|
| | | | |
