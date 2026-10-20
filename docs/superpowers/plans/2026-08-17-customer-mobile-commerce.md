# Customer Mobile Commerce API Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Deliver customer mobile authentication, catalogue, cart, automatic fulfilment fallback, orders, and a production-only OpenAPI contract without building customer UI.

**Architecture:** Store customer identity and attribution globally, use a dedicated seller-neutral customer cart, and create expiring checkout quotes that lock prices before an order transaction resolves distributor-or-admin fulfilment. Preserve existing seller-scoped carts for distributor purchases and assisted sales.

**Tech Stack:** Node 22, TypeScript ESM, Express 5, Prisma 6/PostgreSQL, Zod 4, Vitest, OpenAPI 3.1 YAML.

**Spec:** `nexa-docs/docs/superpowers/specs/2026-08-17-customer-mobile-commerce-design.md`

## Global Constraints

- Swagger contains only `https://nexashopping.duckdns.org/api/v1`; no development or staging server URLs.
- Mobile clients never select or discover distributors.
- One order has one seller; fallback moves the entire order to admin.
- The customer never receives a silent price increase.
- Distributor customer prices stay within the admin-controlled retail range and MRP.
- No customer UI, payments, credit, commissions, referral rewards, GST invoices, or courier integration.

---

### Task 1: Controlled retail pricing schema and rules

**Files:**
- Modify: `nexa-server/prisma/schema.prisma`
- Create: `nexa-server/prisma/migrations/20260817010000_customer_retail_pricing/migration.sql`
- Modify: `nexa-server/src/schemas/inventory.ts`
- Modify: `nexa-server/src/modules/inventory/inventory.service.ts`
- Modify: `nexa-server/src/modules/inventory/inventory.repository.ts`
- Modify: `nexa-server/src/modules/inventory/inventory.service.test.ts`
- Modify: `nexa-admin/src/lib/types.ts`
- Modify: `nexa-admin/src/features/inventory/stock-item-detail.tsx`
- Modify: `nexa-distributor/src/features/inventory/stock-item-detail.tsx`

**Interfaces:**
- Produces: nullable `StockItem.customerPrice` and `StockItem.minimumRetailPrice`; `validateDistributorRetailPrice(owner, variantId, sellPrice, discountPrice)`.
- Consumes: configured primary admin stock, variant MRP, Decimal money helpers.

- [ ] Write failing service tests proving distributor price below floor, above admin customer price, above MRP, and discount above sell price are rejected; boundary values pass.
- [ ] Run `npm test -- src/modules/inventory/inventory.service.test.ts` and confirm expected RED failures.
- [ ] Add migration columns and backfill existing admin rows conservatively with `minimumRetailPrice = sellPrice` and `customerPrice = variant.mrp`; keep nullable at database level for non-admin stock.
- [ ] Extend inventory Zod inputs and views with decimal-string `customerPrice` and `minimumRetailPrice` fields.
- [ ] Implement validation using Decimal; require admin-owned stock to maintain `sellPrice <= minimumRetailPrice <= customerPrice <= mrp`; require distributor effective price to remain within the primary admin range.
- [ ] Update admin pricing UI for wholesale, minimum retail, and customer retail; keep distributor UI limited to customer sell/discount prices.
- [ ] Run server focused/full checks and both panel lint/build commands; commit each repository separately.

---

### Task 2: Customer OTP registration and sessions

**Files:**
- Modify: `nexa-server/prisma/schema.prisma`
- Create: `nexa-server/prisma/migrations/20260817020000_customer_otp_registration/migration.sql`
- Modify: `nexa-server/src/schemas/auth.ts`
- Modify: `nexa-server/src/modules/auth/auth.repository.ts`
- Modify: `nexa-server/src/modules/auth/auth.service.ts`
- Modify: `nexa-server/src/modules/auth/auth.controller.ts`
- Modify: `nexa-server/src/modules/auth/auth.routes.ts`
- Modify: `nexa-server/src/modules/auth/auth.service.test.ts`
- Modify: `nexa-server/src/modules/auth/auth.otp.integration.test.ts`

**Interfaces:**
- Produces: `POST /auth/customer/otp/request` and `POST /auth/customer/otp/verify`.
- Consumes: Ping4SMS adapter, existing hashed OTP challenges, PostgreSQL rate limits, primary admin resolution, distributor referral codes.

- [ ] Write failing tests for new organic registration, referral registration, invalid referral, existing customer login, non-customer phone conflict, expiry, wrong code, single-use, cooldown, and rate limits.
- [ ] Run focused tests and confirm failures occur because customer OTP routes/behavior are missing.
- [ ] Add pending registration fields to the OTP challenge so no customer Account is created before successful verification; store normalized phone and referral intent without logging raw OTP.
- [ ] Implement request with generic anti-enumeration response and the existing approved Ping4SMS route/template.
- [ ] Implement atomic verify: consume challenge, create/reuse ACTIVE customer, set immutable acquisition, create referral relationship when applicable, issue existing access/refresh session.
- [ ] Run focused integration tests, then full server lint/typecheck/test/build; commit.

---

### Task 3: Customer profile and addresses API

**Files:**
- Create: `nexa-server/src/schemas/customer-profile.ts`
- Create: `nexa-server/src/modules/customer-profile/customer-profile.repository.ts`
- Create: `nexa-server/src/modules/customer-profile/customer-profile.service.ts`
- Create: `nexa-server/src/modules/customer-profile/customer-profile.controller.ts`
- Create: `nexa-server/src/modules/customer-profile/customer-profile.routes.ts`
- Create: `nexa-server/src/modules/customer-profile/customer-profile.service.test.ts`
- Modify: `nexa-server/src/modules/registry.ts`

**Interfaces:**
- Produces: `GET/PATCH /customer/profile` and CRUD `/customer/addresses` endpoints.
- Consumes: authenticated CUSTOMER account and existing `Address` model.

- [ ] Write failing tests for self-only profile read/update, address ownership, one default address, invalid pincode/phone, and deleting the default address.
- [ ] Run focused tests for RED.
- [ ] Implement strict Zod schemas, repository scope by `accountId`, service rules, thin controllers, and route registration.
- [ ] Run focused and full server verification; commit.

---

### Task 4: Customer global catalogue API

**Files:**
- Create: `nexa-server/src/schemas/storefront.ts`
- Create: `nexa-server/src/modules/storefront/storefront.repository.ts`
- Create: `nexa-server/src/modules/storefront/storefront.service.ts`
- Create: `nexa-server/src/modules/storefront/storefront.controller.ts`
- Create: `nexa-server/src/modules/storefront/storefront.routes.ts`
- Create: `nexa-server/src/modules/storefront/storefront.service.test.ts`
- Modify: `nexa-server/src/modules/registry.ts`

**Interfaces:**
- Produces: `GET /storefront/products` and `GET /storefront/products/:id` for CUSTOMER accounts.
- Consumes: global published catalogue, customer acquisition, attributed distributor stock, primary admin retail stock.

- [ ] Write failing tests proving every published product is visible independent of distributor stock, no distributor list/identity is returned, referral customers receive a valid distributor offer when possible, and organic customers receive admin retail price.
- [ ] Run focused tests for RED.
- [ ] Implement cursor pagination/search/category filters and resolved customer offer fields `{ price, discountPrice, available }` without seller-selection data.
- [ ] Ensure unavailable distributor stock does not hide a globally published product; use admin availability as fallback indicator.
- [ ] Run focused/full verification; commit.

---

### Task 5: Seller-neutral customer cart and quotes

**Files:**
- Modify: `nexa-server/prisma/schema.prisma`
- Create: `nexa-server/prisma/migrations/20260817030000_customer_cart_quotes/migration.sql`
- Create: `nexa-server/src/schemas/customer-cart.ts`
- Create: `nexa-server/src/modules/customer-cart/customer-cart.repository.ts`
- Create: `nexa-server/src/modules/customer-cart/customer-cart.service.ts`
- Create: `nexa-server/src/modules/customer-cart/customer-cart.controller.ts`
- Create: `nexa-server/src/modules/customer-cart/customer-cart.routes.ts`
- Create: `nexa-server/src/modules/customer-cart/customer-cart.service.test.ts`
- Modify: `nexa-server/src/modules/registry.ts`

**Interfaces:**
- Produces: `CustomerCart`, `CustomerCartItem`, `CheckoutQuote`, `CheckoutQuoteItem`; `GET /customer/cart`, item POST/PATCH/DELETE, and `POST /customer/cart/quote`.
- Consumes: customer attribution, distributor/admin stock, controlled price range, core pricing.

- [ ] Write failing tests for global cart lines, complete distributor fulfilment, one missing distributor line causing complete admin fallback, no split, exact unavailable lines, and quote price/expiry snapshots.
- [ ] Run focused tests for RED.
- [ ] Add models with one cart per customer, unique variant lines, quote seller/attribution/fallback metadata, price snapshots, and expiry index.
- [ ] Implement mutations that never accept `sellerAccountId`; implement resolver in canonical order: attributed distributor complete cart, then primary admin complete cart, else unavailable.
- [ ] Implement a 10-minute quote; quote prices are distributor prices when distributor fulfils and admin customer prices when initial fallback is already required.
- [ ] Run focused/full verification; commit.

---

### Task 6: Customer order placement and full-order fallback

**Files:**
- Modify: `nexa-server/prisma/schema.prisma`
- Create: `nexa-server/prisma/migrations/20260817040000_order_attribution_fallback/migration.sql`
- Modify: `nexa-server/src/schemas/orders.ts`
- Modify: `nexa-server/src/modules/orders/orders.repository.ts`
- Modify: `nexa-server/src/modules/orders/orders.service.ts`
- Modify: `nexa-server/src/modules/orders/orders.controller.ts`
- Modify: `nexa-server/src/modules/orders/orders.routes.ts`
- Modify: `nexa-server/src/modules/orders/orders.service.test.ts`

**Interfaces:**
- Produces: `POST /customer/orders`, customer-scoped list/detail/cancel, order attribution and fallback snapshot fields.
- Consumes: valid checkout quote, existing inventory reservation transaction, order state machine.

- [ ] Write failing tests for distributor fulfilment, distributor stock loss causing entire admin fallback at quoted prices, absence of split orders, admin insufficiency rollback, expired/used quote rejection, and customer-only order access.
- [ ] Run focused tests for RED.
- [ ] Add nullable order attribution/fallback columns and indexes.
- [ ] Implement one transaction that locks/consumes the quote, rechecks complete stock, chooses quoted distributor or full admin fallback, reserves exactly one seller, snapshots prices/address/attribution, and clears the customer cart.
- [ ] Preserve existing distributor purchase and assisted-order behavior unchanged.
- [ ] Run order tests, inventory concurrency tests, and full verification; commit.

---

### Task 7: Production-only customer OpenAPI contract

**Files:**
- Create: `nexa-server/docs/openapi/customer-mobile.yaml`
- Create: `nexa-server/scripts/validate-customer-openapi.mjs`
- Modify: `nexa-server/package.json`
- Modify: `nexa-docs/docs/API.md`

**Interfaces:**
- Produces: OpenAPI 3.1 customer contract and `npm run openapi:validate`.
- Consumes: implemented customer routes and stable error envelope.

- [ ] Document only customer auth, profile, address, storefront, cart, quote, order, cancellation, tracking, and shared health/readiness endpoints required by mobile integration.
- [ ] Set the sole server URL to `https://nexashopping.duckdns.org/api/v1`.
- [ ] Define bearer authentication, reusable request/response/error/pagination schemas, every status code, and realistic examples with no secrets or live personal data.
- [ ] Add validation that parses the YAML, rejects any non-production server, verifies unique operation IDs, and checks documented paths against an exported route manifest.
- [ ] Run `npm run openapi:validate`, server full verification, and manually import the YAML into a compatible Swagger/Postman viewer; commit documentation and validator in their owning locations.

---

### Task 8: Release verification and Docker deployment

**Files:**
- Modify: `nexa-docs/docs/ROADMAP.md`
- Modify: `nexa-server/DEPLOY.md` only if `deploy.sh` needs an actual change.

**Interfaces:**
- Produces: deployable customer API and mobile-team handoff.
- Consumes: Tasks 1-7 plus the distributor customers/assisted-sales plan.

- [ ] Generate Prisma, deploy migrations to the local Docker PostgreSQL database, and run server lint/typecheck/all tests/build/OpenAPI validation.
- [ ] Run admin and distributor panel lint/build checks.
- [ ] Exercise real OTP, organic registration, referral registration, distributor fulfilment, full admin fallback, no-stock failure, cancellation, and privacy boundaries against a local production-like stack.
- [ ] Confirm the Swagger examples match real responses and contain only the production URL.
- [ ] Commit and push only verified repository changes; report exact hashes and any local `nexa-docs` files that cannot be pushed because that directory has no Git metadata.
- [ ] Deploy server exclusively with `git pull origin main` followed by `./deploy.sh`; verify `/healthz`, `/readyz`, logs, and one real mobile-style customer flow.
