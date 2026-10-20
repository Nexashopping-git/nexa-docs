# ROADMAP

> **Historical phase plan (2026-10-20):** The phase checklist below is not a live completion tracker. See [the repository overview](../README.md) and current application repositories for implemented features. The diary cash-receipt feature was added after this plan.

Build order is deliberate: **foundation → identity → catalog → money → panels**. Each phase has
an exit criterion. We do not start phase N+1 until phase N's criterion is met, because every
shortcut here is what v1 paid for later.

At the start of a session, state which phase and task you're on.

---

## Phase 0 — Foundation (no features)

**Unblocked.** Greenfield DB (ADR-0003), Next.js panels (ADR-0004).
Step-by-step task list: `docs/PHASE-0.md`.

1. GitHub org, five repos created, branch protection
2. `nexa-docs` pushed — this instruction set plus an empty `API.md`
3. `nexa-server` skeleton: prisma + identity schema, `core/`, `/healthz`, module registry
4. Docker: dev compose (postgres + redis), `Dockerfile`, prod compose with Caddy
5. Three Next.js shells deploying to Vercel
6. CI in every repo: lint, typecheck, test, migrate-diff

**Exit:** the API boots against dockerised Postgres and `/healthz` returns 200; three Vercel
projects deploy their shells and one is proven to call the live API from a browser; CI green.

---

## Phase 1 — Identity & access

1. `Account`, profiles, `Session`, `DeviceToken` models + migration
2. Firebase phone-OTP verification → session issue; refresh rotation with reuse detection
3. `authenticate` + `authorize` middleware, permission table in code
4. Accounts module: profile read/update, admin list/search accounts, co-admin creation,
   distributor onboarding + referral link
5. Permission matrix test: every role × every endpoint

**Exit:** all three roles can log in, obtain a session, refresh it, log out; the permission
matrix test passes; no role logic exists outside `core/auth` and `*.policy.ts`.

---

## Phase 2 — Catalog

**Unblocked.** Variants are first-class (ADR-0005).

1. Category tree, attribute definitions, product, media, variants
2. Variant matrix generation from variant-axis attributes; auto-created single variant for
   products with no axes, so there is never a "product without a variant" branch
3. Admin CRUD; public read endpoints with cursor pagination, filters, sort
4. Search (tsvector or trigram), media upload via signed URLs
5. Seed script with realistic sample catalog, including multi-variant products

**Exit:** admin can create a product with three variants, media, and attributes; public listing
paginates, filters, and searches; listing p95 under 200 ms on 10k variants with an EXPLAIN
attached to the PR.

---

## Phase 3 — Inventory

1. `StockItem` (keyed on `variantId`) + `StockLedger`; `applyDelta()` as the single writer
2. Per-owner price lists; listing/unlisting
3. Reserve / release semantics with row locking
4. Admin → distributor stock transfer as one transaction (TRANSFER_OUT + TRANSFER_IN)
5. Low-stock alerts

**Exit:** a concurrency test proves stock cannot go negative under parallel reservations;
ledger sum equals `onHand` for every item in a property test.

---

## Phase 4 — Cart & orders

1. Cart lifecycle, live re-pricing on read
2. Pricing engine: line price, discount, tax, totals — pure, fully unit-tested
3. Order placement transaction: reserve → ledger → order + items → snapshot address
4. State machine + `OrderEvent` audit trail; cancellation with stock release
5. Distributor-assisted order creation (`channel = DISTRIBUTOR_ASSISTED`)
6. Order expiry job for unpaid orders

**Exit:** an order placed by each channel produces correct totals, correct stock ledger
entries, and a complete event trail; cancelling restores stock exactly.

---

## Phase 5 — Payments

**Blocked on:** Q5 (gateway).

1. `PaymentIntent` + idempotency keys; provider behind an interface
2. PhonePe adapter: initiate, verify, callback
3. Webhook handler with `providerEventId` dedupe and signature verification
4. Refunds; reconciliation job comparing gateway status to our records

**Exit:** replaying the same webhook 10× moves money exactly once (test); a failed payment
leaves order and stock in a consistent state; reconciliation reports zero mismatches on
seeded data.

---

## Phase 6 — Credit

**Blocked on:** Q4 (credit terms).

1. `CreditAccount` + `CreditEntry`; `credit.service.post()` as the single writer
2. Limit allocation/revocation by admin, with audit
3. Order-on-credit: charge at placement, respect available credit, lock the account row
4. Repayment via payment intent; allocation order for partial repayment
5. Nightly balance re-derivation + drift alert; distributor exposure report

**Exit:** concurrent orders cannot exceed the credit limit (test); ledger sum equals
`currentBalance` for every account after a randomised transaction property test.

---

## Phase 7 — Admin panel

1. Next.js app, auth flow, layout, base components
2. Dashboard, catalog management, inventory, orders, credit management, accounts/co-admins,
   banners

KYC review and notifications are dropped from this version's scope (ADR-0012, ADR-0020).

**Exit:** every admin capability that exists in v1 is available in v2, with loading/empty/error
states on every screen.

---

## Phase 8 — Distributor panel

Dashboard, catalog + purchase from admin, own inventory & pricing, customers, assisted order
creation, billing/invoice PDF, credit & repayment.

KYC submission is dropped from this version's scope (ADR-0020).

**Exit:** a distributor can complete the full loop — buy stock on credit, list it, create a
customer order, collect payment, repay admin — end to end on staging.

---

## Phase 9 — Supporting

Notifications (FCM fan-out, templates, read state), reviews & ratings with moderation, banners
and content, reporting/exports, marketing site with legal pages.

---

## Phase 10 — Hardening & cutover

Load test the hot paths · finalise indexes from real query plans · backup + documented restore
drill · alerting on error rate, latency, credit drift, failed payments · runbook ·
staging soak · data migration rehearsal (if Q1 = migrate) · cutover with rollback plan.

---

## Customer mobile commerce — implemented API foundation

- Global phone-based customer identity with immutable admin/distributor acquisition
- Customer OTP login, profile, and private saved addresses
- Shared catalog with controlled retail pricing and no distributor-selection UI
- Persistent cart and ten-minute price-lock quote
- Whole-cart distributor fulfilment with automatic admin fallback; never split an order
- Customer-safe order placement, history, detail, and cancellation
- Customer-only OpenAPI 3.1 contract using the production URL

Payments, commissions/rewards, GST invoices, and courier integration remain separate future
work and are intentionally absent from this release.

---

## Working rules

- Server module before its panel screens — the contract must exist first.
- No feature is "done" without its tests and its doc update.
- If a phase reveals a schema mistake, fix the schema and update `docs/DATA-MODEL.md` in the
  same PR. Do not work around a bad model.
- Anything not in this roadmap is out of scope until it's added to this roadmap.
