# Distributor Customers and Assisted Sales Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let distributors maintain private customer relationships and complete assisted sales from their own inventory.

**Architecture:** Keep the global customer identity unique by phone, store immutable acquisition in `CustomerProfile`, and store distributor-private CRM data in a many-to-many `DistributorCustomer` relation. Add a focused server `customers` module, then reuse the existing cart, order, inventory, and fulfilment contracts from new distributor-panel screens.

**Tech Stack:** Node 22, TypeScript ESM, Express 5, Prisma 6/PostgreSQL, Zod 4, Vitest, Next.js 16 App Router, React 19, TanStack Query 5.

**Spec:** `nexa-docs/docs/superpowers/specs/2026-08-17-distributor-customers-assisted-sales-design.md`

## Global Constraints

- One global `CUSTOMER` account per E.164 phone number.
- Acquisition attribution is immutable after the customer is created.
- Distributor-private relationship data is visible only to that distributor or an admin.
- Controllers remain thin; Prisma is used only in `*.repository.ts` files.
- Money and inventory continue through the existing pricing, cart, order, and inventory services.
- Do not add customer OTP login, pincode routing, payments, credit, referral rewards, or GST invoices in this release.
- Every API shape change updates `nexa-docs/docs/API.md` and the matching Zod schema in the same task.

---

### Task 1: Customer identity and relationship schema

**Files:**
- Modify: `nexa-server/prisma/schema.prisma`
- Create: `nexa-server/prisma/migrations/20260817000000_distributor_customers/migration.sql`
- Modify: `nexa-docs/docs/DATA-MODEL.md`

**Interfaces:**
- Produces: Prisma models `CustomerProfile` and `DistributorCustomer` and enums `CustomerAcquisitionSource`, `DistributorCustomerStatus`.
- Consumes: existing `Account`, `Role`, and account self-relations.

- [ ] **Step 1: Add the Prisma schema models**

Define `CustomerProfile` with unique `accountId`, `acquisitionSource`, nullable `acquiredByAccountId`, and `acquiredAt`; add account relations named `CustomerIdentity`, `CustomerAcquirer`, `DistributorCustomerOwner`, and `DistributorCustomerIdentity`. Define `DistributorCustomer` with UUID `id`, distributor/customer UUIDs, nullable `displayName` and `notes`, `ACTIVE|BLOCKED` status, timestamps, unique `(distributorAccountId, customerAccountId)`, and indexes for distributor/status/createdAt and customer.

- [ ] **Step 2: Generate and inspect the migration**

Run:

```powershell
$env:DATABASE_URL='postgresql://<USER>:<PASSWORD>@localhost:5432/nexa?schema=public'
npx prisma migrate dev --name distributor_customers --create-only
```

Expected: SQL creates both enums, both tables, unique constraints, indexes, and `Restrict` account foreign keys. Rename the generated migration directory to `20260817000000_distributor_customers` if Prisma uses a different timestamp.

- [ ] **Step 3: Validate and regenerate Prisma**

Run:

```powershell
npx prisma validate
npx prisma generate
```

Expected: both commands exit 0.

- [ ] **Step 4: Document the model**

Update `DATA-MODEL.md` to state that customer identity is global, acquisition is immutable, distributor CRM data is relationship-scoped, and a customer may relate to multiple distributors.

- [ ] **Step 5: Commit the schema deliverable**

```bash
git add prisma/schema.prisma prisma/migrations/20260817000000_distributor_customers/migration.sql
git commit -m "feat(customers): add distributor customer relationships"
```

---

### Task 2: Customer-management API

**Files:**
- Create: `nexa-server/src/schemas/customers.ts`
- Create: `nexa-server/src/modules/customers/customers.repository.ts`
- Create: `nexa-server/src/modules/customers/customers.service.ts`
- Create: `nexa-server/src/modules/customers/customers.controller.ts`
- Create: `nexa-server/src/modules/customers/customers.routes.ts`
- Create: `nexa-server/src/modules/customers/customers.service.test.ts`
- Modify: `nexa-server/src/modules/registry.ts`
- Modify: `nexa-docs/docs/API.md`

**Interfaces:**
- Produces: `POST /api/v1/customers`, `GET /api/v1/customers`, `GET /api/v1/customers/:id`, and `PATCH /api/v1/customers/:id`.
- Produces: `CustomerRelationshipView = { id, customer: { id, name, phone, status }, displayName, notes, status, acquisitionSource, acquiredByAccountId, createdAt, updatedAt }`.
- Consumes: authenticated `Account`, `decodeCursor`, `paginate`, `withTransaction`, and standard HTTP errors.

- [ ] **Step 1: Write failing service tests**

Cover these cases in `customers.service.test.ts` using mocked repository functions:

```ts
it("creates a CUSTOMER and assisted attribution when the phone is new")
it("reuses an existing CUSTOMER without changing acquisition")
it("returns the existing relationship for an idempotent repeat")
it("rejects a phone used by a non-customer account")
it("scopes list, detail, and update to the actor distributor")
it("rejects suspended distributors")
```

- [ ] **Step 2: Run the tests and confirm failure**

Run:

```bash
npm test -- src/modules/customers/customers.service.test.ts
```

Expected: FAIL because the customers module does not exist.

- [ ] **Step 3: Define strict request schemas**

Implement:

```ts
createCustomerSchema = { phone: E.164, name: non-empty, displayName?: string, notes?: string }
listCustomersQuerySchema = { q?: string, status?: "ACTIVE"|"BLOCKED", cursor?: string, limit: 1..100 default 20 }
updateCustomerRelationshipSchema = { displayName?: string|null, notes?: string|null, status?: "ACTIVE"|"BLOCKED" }
customerIdParamSchema = { id: uuid }
```

- [ ] **Step 4: Implement atomic repository operations**

Implement `attachCustomer(distributorAccountId, input)` as one `withTransaction()` operation: find account by phone; create an ACTIVE `CUSTOMER` plus `CustomerProfile(DISTRIBUTOR_ASSISTED, distributor)` when absent; reject any existing non-customer; then upsert the unique relationship without overwriting existing private fields. Add scoped list/detail/update repository functions that always include `distributorAccountId` in their `where` clause.

- [ ] **Step 5: Implement service authorization and pagination**

Require `actor.role === "DISTRIBUTOR" && actor.status === "ACTIVE"`, with admin access allowed only when an explicit distributor scope is later designed; for this release keep routes distributor-only. Return `ConflictError` for non-customer phone collisions and `NotFoundError` for relationships outside the actor's scope.

- [ ] **Step 6: Implement controllers, routes, and registry mounting**

Mount the router once at `/api/v1/customers`. Serialize dates to ISO strings and return `201` only when a new relationship is created; return `200` for idempotent repeats.

- [ ] **Step 7: Run focused tests**

```bash
npm test -- src/modules/customers/customers.service.test.ts
```

Expected: all customer service tests pass.

- [ ] **Step 8: Document exact API contracts**

Add the four endpoints, request/response examples, authorization, pagination, idempotency, conflict behavior, and immutable-attribution rules to `API.md`.

- [ ] **Step 9: Run server quality checks and commit**

```bash
npm run lint
npm run typecheck
npm test -- --run
npm run build
git add src/modules/customers src/schemas/customers.ts src/modules/registry.ts
git commit -m "feat(customers): add distributor customer API"
```

Expected: lint, typecheck, all tests, and build exit 0.

---

### Task 3: Assisted-sale contract tests and client hooks

**Files:**
- Modify: `nexa-server/src/modules/orders/orders.service.test.ts`
- Modify: `nexa-server/src/modules/cart/cart.service.test.ts` if absent create it beside `cart.service.ts`
- Modify: `nexa-distributor/src/lib/types.ts`
- Create: `nexa-distributor/src/features/customers/api.ts`
- Modify: `nexa-distributor/src/features/cart/api.ts`
- Modify: `nexa-distributor/src/features/orders/api.ts`

**Interfaces:**
- Produces: `useCustomers`, `useCustomer`, `useCreateCustomer`, `useUpdateCustomer`, buyer-aware cart hooks, `usePlaceAssistedOrder`, `useSales`, `useConfirmOrder`, `useShipOrder`, and `useDeliverOrder`.
- Consumes: existing `/customers`, `/cart`, `/orders`, and inventory contracts.

- [ ] **Step 1: Add server regression tests for the existing assisted path**

Test that a distributor seller can build another customer's cart, place only with `channel: "DISTRIBUTOR_ASSISTED"` plus `buyerAccountId`, cannot assist for another seller, and cannot place for a non-customer buyer. If the non-customer role guard is missing, add it to `orders.service.ts` through its own repository lookup.

- [ ] **Step 2: Run focused server tests**

```bash
npm test -- src/modules/cart/cart.service.test.ts src/modules/orders/orders.service.test.ts
```

Expected: all assisted-flow tests pass after any required role guard is implemented.

- [ ] **Step 3: Add panel response and request types**

Add `CustomerRelationship`, `CustomerAcquisitionSource`, `DistributorCustomerStatus`, `CreateCustomerBody`, and `UpdateCustomerBody` matching `API.md`. Change `PlaceOrderBody.channel` to `"WEB" | "DISTRIBUTOR_ASSISTED"` and add optional `buyerAccountId`.

- [ ] **Step 4: Make cart hooks buyer-aware without breaking purchases**

Change signatures to:

```ts
useCart(sellerAccountId: string, buyerAccountId?: string)
useAddToCart(sellerAccountId: string, buyerAccountId?: string)
useUpdateCartItem(sellerAccountId: string, buyerAccountId?: string)
useRemoveCartItem(sellerAccountId: string, buyerAccountId?: string)
```

Use query keys `['cart', sellerAccountId, buyerAccountId ?? 'self']` and include `buyerAccountId` only on GET/POST requests. Existing buy-from-admin calls remain valid without a second argument.

- [ ] **Step 5: Add customer and seller-order hooks**

Implement paginated customer search and mutations. Add `useSales(filters, cursor)` using `role=seller`; add confirm/ship/deliver mutations that invalidate `['sales']`, `['orders']`, `['order', id]`, and `['my-inventory']` after delivery.

- [ ] **Step 6: Verify both repositories and commit separately**

```bash
cd ../nexa-server && npm run lint && npm run typecheck && npm test -- --run
cd ../nexa-distributor && npm run lint && npm run build
```

Expected: all commands exit 0. Commit server regression changes in `nexa-server`, then client contract changes in `nexa-distributor` with `feat(customers): add assisted sales client contracts`.

---

### Task 4: Distributor customer-management screens

**Files:**
- Create: `nexa-distributor/src/app/dashboard/customers/page.tsx`
- Create: `nexa-distributor/src/app/dashboard/customers/[id]/page.tsx`
- Create: `nexa-distributor/src/features/customers/customer-form.tsx`
- Create: `nexa-distributor/src/features/customers/customer-detail.tsx`
- Modify: `nexa-distributor/src/app/dashboard/layout.tsx`

**Interfaces:**
- Produces: customer list/create UI and `/dashboard/customers/:id` relationship detail with a `Start sale` link.
- Consumes: customer hooks from Task 3 and shared `Button`, `Input`, `Select`, `Card`, `EmptyState`, `ErrorState`, and `Spinner`.

- [ ] **Step 1: Add Customers navigation and list page**

Add a Customers item to `NAV`. Build paginated search with loading, empty, error, and retry states. Display relationship name, phone, status, acquisition source, and relationship date without exposing any other distributor data.

- [ ] **Step 2: Add create/attach form**

Collect E.164 phone, name, optional local display name, and notes. On `CONFLICT`, display the server message; on success navigate to the returned relationship detail. Disable duplicate submissions while pending.

- [ ] **Step 3: Add scoped detail and edit UI**

Allow editing only `displayName`, `notes`, and relationship status. Display immutable acquisition information read-only. Link `Start sale` to `/dashboard/customers/{id}/sale`.

- [ ] **Step 4: Verify responsive and error states**

Run:

```bash
npm run lint
npm run build
```

Expected: both exit 0; inspect list, empty, creation, conflict, and detail states at mobile and desktop widths.

- [ ] **Step 5: Commit**

```bash
git add src/app/dashboard/customers src/features/customers src/app/dashboard/layout.tsx
git commit -m "feat(customers): add distributor customer management"
```

---

### Task 5: Assisted cart and checkout screens

**Files:**
- Create: `nexa-distributor/src/app/dashboard/customers/[id]/sale/page.tsx`
- Create: `nexa-distributor/src/features/customers/assisted-sale.tsx`
- Create: `nexa-distributor/src/features/customers/assisted-checkout.tsx`

**Interfaces:**
- Produces: customer-scoped inventory selection, cart editing, address capture, and assisted order placement.
- Consumes: selected `CustomerRelationship`, `useInventory`, buyer-aware cart hooks, and `usePlaceAssistedOrder`.

- [ ] **Step 1: Build customer-scoped product selection**

Load the relationship first. Show the distributor's own items filtered client-side to `isListed && available > 0`, with search, price, available quantity, and add-to-cart. Pass `sellerAccountId = authenticated distributor id` and `buyerAccountId = customer.id` to every cart mutation.

- [ ] **Step 2: Build cart editing**

Show live repriced totals and available quantity. Support absolute quantity update and removal. Keep the query key customer-scoped so carts for two customers never share cached data.

- [ ] **Step 3: Build checkout address form**

Collect `contactName`, E.164 `contactPhone`, `line1`, optional `line2`, `city`, `state`, six-digit Indian `pincode`, and default country `IN`. Do not store the address globally in this release; send it as the order snapshot.

- [ ] **Step 4: Place the assisted order**

Send:

```ts
{
  sellerAccountId: distributorAccountId,
  buyerAccountId: customer.customer.id,
  channel: "DISTRIBUTOR_ASSISTED",
  shippingAddress
}
```

On success, clear/invalidate the scoped cart and navigate to `/dashboard/sales/{order.id}`. Display insufficient-stock and business-rule errors without retrying the mutation automatically.

- [ ] **Step 5: Verify and commit**

```bash
npm run lint
npm run build
git add src/app/dashboard/customers src/features/customers
git commit -m "feat(sales): add distributor assisted checkout"
```

Expected: lint and build exit 0.

---

### Task 6: Distributor sales and fulfilment screens

**Files:**
- Create: `nexa-distributor/src/app/dashboard/sales/page.tsx`
- Create: `nexa-distributor/src/app/dashboard/sales/[id]/page.tsx`
- Create: `nexa-distributor/src/features/orders/sales-order-detail.tsx`
- Modify: `nexa-distributor/src/app/dashboard/layout.tsx`
- Modify: `nexa-distributor/src/app/dashboard/page.tsx`

**Interfaces:**
- Produces: seller-side order list, detail, and legal state transitions.
- Consumes: seller-order hooks from Task 3 and the existing `Order` state machine.

- [ ] **Step 1: Add Sales navigation and paginated list**

Use `GET /orders?role=seller`, status filters, and cursor pagination. Clearly label these as customer sales so they cannot be confused with `/dashboard/orders` purchases from admin.

- [ ] **Step 2: Add sales detail**

Show order number, customer delivery snapshot, items, totals, timestamps, payment status, fulfilment status, and cancellation reason.

- [ ] **Step 3: Add state-machine actions**

Render only these actions:

```text
AWAITING_PAYMENT -> Confirm or Cancel
CONFIRMED       -> Ship or Cancel
SHIPPED         -> Deliver
DELIVERED       -> no mutation
CANCELLED       -> no mutation
```

Require a confirmation dialog for each mutation, disable actions while pending, and render the server error if the order changed concurrently.

- [ ] **Step 4: Add dashboard entry points**

Add cards linking to Customers and Sales. Do not invent sales totals until a dedicated summary endpoint exists.

- [ ] **Step 5: Verify and commit**

```bash
npm run lint
npm run build
git add src/app/dashboard/sales src/features/orders/sales-order-detail.tsx src/app/dashboard/layout.tsx src/app/dashboard/page.tsx
git commit -m "feat(sales): add distributor fulfilment workflow"
```

Expected: lint and build exit 0.

---

### Task 7: Cross-repository verification and deployment handoff

**Files:**
- Modify: `nexa-docs/docs/ROADMAP.md`
- Modify: `nexa-docs/docs/API.md` only if verification finds contract drift
- Modify: `nexa-server/DEPLOY.md` only if the existing `deploy.sh` cannot deploy the migration unchanged

**Interfaces:**
- Produces: verified release and exact Docker deployment instructions.
- Consumes: all earlier tasks.

- [ ] **Step 1: Run final server verification from a freshly generated client**

```powershell
cd C:\Users\satya\Desktop\nexashopping\nexa-server
$env:DATABASE_URL='postgresql://<USER>:<PASSWORD>@localhost:5432/nexa?schema=public'
npx prisma generate
npx prisma migrate deploy
npm run lint
npm run typecheck
npm test -- --run
npm run build
```

Expected: migration applies once; all remaining commands exit 0.

- [ ] **Step 2: Run final distributor-panel verification**

```powershell
cd C:\Users\satya\Desktop\nexashopping\nexa-distributor
npm run lint
npm run build
```

Expected: both exit 0.

- [ ] **Step 3: Perform the real end-to-end acceptance test**

Using an active distributor: attach a new phone; repeat attachment and confirm the same relationship returns; add inventory to that customer's cart; place an assisted order; confirm, ship, and deliver; verify seller `onHand` decreases only on delivery and the stock ledger contains `RESERVE`, `RELEASE`, and `SALE_OUT`; verify another distributor cannot read that relationship.

- [ ] **Step 4: Update roadmap status**

Mark distributor customers and assisted ordering implemented while leaving customer self-registration, pincode distributor discovery, admin fallback, payments, credit, and invoices explicitly pending.

- [ ] **Step 5: Push each repository intentionally**

Confirm `git status --short` contains no unrelated files, push `nexa-server` and `nexa-distributor` only after their respective verification passes, and report the exact commit hashes. If `nexa-docs` still has no Git metadata, report its local documentation changes rather than claiming they were pushed.

- [ ] **Step 6: Deploy through the repository script**

On the server:

```bash
cd /path/to/nexa-server
git pull origin main
./deploy.sh
```

Expected: Docker builds the API, `prisma migrate deploy` applies the customer migration, containers become healthy, and `/healthz` plus `/readyz` return success. No PM2 or host-side build commands are used.
