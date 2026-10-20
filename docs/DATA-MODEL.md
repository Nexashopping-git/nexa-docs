# DATA-MODEL

> **Historical design note (2026-10-20):** The implemented Prisma source is `nexa-server/prisma/schema.prisma`. This design predates `DiaryEntry` and `DiaryAuditEvent`; see the [diary API contract](API.md#diary-cash-receipt-ledger-october-2026) and server migration `20261017000000_diary_cash_entries` for the cash-receipt model.

Status: **v0.2** — identity, catalog, and inventory are settled and implementable
(ADR-0003/0004/0005). Orders, payments, and credit are still design sketches pending Q4, Q5, Q9.
Concrete Prisma for the settled parts: `docs/schema/core.prisma`.

The database is greenfield — v1 is shut down and backed up, so nothing here is constrained by
the old shapes. Any historical import is a separate script written after the schema is stable.

---

## 1. Principles

1. **One entity, one table.** If two tables differ only by who owns the row, they are one
   table with an owner column.
2. **Explicit foreign keys.** No `entityId` + `entityType` pairs. If a column can point at two
   different things, model the two things properly or split the table.
3. **Money is `Decimal(14,2)`** in the database and a `Money` value object in code.
   Never `Float`. Never JS `number` arithmetic on money.
4. **Historical records are snapshots.** Orders, invoices, and ledger rows copy the values they
   depend on. They never join to mutable rows to reconstruct the past.
5. **Financial truth is append-only.** Balances are maintained only alongside a ledger insert,
   inside one transaction, and are re-derivable by summing the ledger.
6. **Nothing important is hard-deleted.** `deactivatedAt` / `archivedAt`, and FK deletes are
   `Restrict` by default. `Cascade` is allowed only for rows that are meaningless without the
   parent (cart items, attribute values, device tokens) and must be justified in a comment.
7. **Every FK is indexed**, plus composite indexes matching real query paths.

---

## 2. Identity

```
Account            one row per human/org. phone (unique), email, name, status, role
├── AdminProfile        admin-only fields
├── DistributorProfile  referralCode, referredByAccountId (real FK), tier, territory
└── CustomerProfile     acquisitionSource, acquiredByAccountId, acquiredAt

DistributorCustomer     distributorAccountId, customerAccountId, displayName, notes, status
                        unique(distributorAccountId, customerAccountId)
```

**Changes from v1**
- `User` and `Distributor` no longer duplicate `creditScore` / `referredBy`. Credit belongs to
  the credit module; referral is one FK.
- `Referral` as a separate table is dropped. `DistributorProfile.referredByAccountId` is the
  single source of truth. A referral *event* (with reward) is a ledger row if we ever pay for
  referrals — not a duplicate of the edge.
  > `[DECISION NEEDED]` Q6: do referrals pay out? If yes we need `ReferralReward`.
- Profile tables are kept only if they hold ≥3 role-specific fields; otherwise the fields go on
  `Account`. Do not create an empty profile table "for symmetry".
- A phone identifies one global customer account. Acquisition is immutable; connecting the same
  customer to another distributor never overwrites the original `ORGANIC`, `REFERRAL`, or
  `DISTRIBUTOR_ASSISTED` source. `DistributorCustomer` stores each distributor's private CRM link.

**Sessions** — v1 stored refresh tokens and FCM tokens in one `Token` table with a `type`
discriminator, and encrypted the JWT payload with RSA. v2:

```
Session      id, accountId, refreshTokenHash, deviceInfo, ip, expiresAt, revokedAt
DeviceToken  id, accountId, fcmToken, platform, lastSeenAt
```
Access token: short-lived JWT, signed, carrying `sub` + `role` + `sessionId` and nothing
sensitive. Refresh token: opaque random, **hashed** at rest, rotated on use, reuse-detection
revokes the session family. One cookie name for everyone.

---

## 3. Catalog

`ProductVariant` is the sellable unit. Every product has **at least one** variant — a product
with no variant axes gets one auto-created on save, so no query and no UI ever has to branch on
"does this product have variants".

```
Category           id, name, slug, parentId?, position, isActive
AttributeDef       id, categoryId, name, code, type, unit?,
                   isVariantAxis, isFilterable, position
AttributeOption    id, attributeDefId, value, position       -- for enum-type axes
Product            id, categoryId, brand, name, slug, description, shortDescription,
                   status(DRAFT|ACTIVE|ARCHIVED), archivedAt
ProductMedia       id, productId, variantId?, url, alt, position, isPrimary
ProductVariant     id, productId, sku (unique), name, mrp, isActive, archivedAt
                   -- mrp is the printed/list price for display only; the selling price
                   -- lives on StockItem, because each owner prices independently
VariantOption      id, variantId, attributeDefId, value      -- one row per axis
                   @@unique([variantId, attributeDefId])
ProductAttribute   id, productId, attributeDefId, value      -- descriptive, non-axis
```

**How a variant is defined.** Axes come from `AttributeDef.isVariantAxis` on the product's
category — e.g. *Size* and *Colour*. A variant is a specific combination, stored as
`VariantOption` rows. The combination is enforced unique per product by a check in the service
(Postgres can't express it as a simple constraint), and the SKU is globally unique.

Why rows rather than a JSONB blob on the variant: we need to filter listings by axis
("show me 500 ml") and that's an index on `VariantOption(attributeDefId, value)` rather than a
JSONB scan.

**Changes from v1**
- `imageUrls String[]` becomes `ProductMedia`, with ordering, alt text, a primary flag, and an
  optional `variantId` so a colour variant can show its own photo.
- `Category → Product` becomes `onDelete: Restrict`. Categories deactivate, never delete.
- Product carries **no selling price**. Admin and each distributor price the same variant
  differently — this is precisely why v1 ended up with two inventory tables.
- `slug` on product and category for storefront URLs.
- Products are archived, never deleted, because order items reference variants.

## 4. Inventory — the big consolidation

v1: `AdminInventory`, `DistributorInventory`, `AdminInventoryLog`, `DistributorInventoryLog`.
v2: **two tables**.

```
StockItem     id, ownerAccountId, variantId, onHand, reserved,
              available (generated column: onHand - reserved),
              sellPrice, discountPrice, minimumRetailPrice?, customerPrice?, taxRatePct, isListed
              @@unique([ownerAccountId, variantId])
              -- keyed on variant, never product: "product stock" is an aggregate

StockLedger   id, stockItemId, delta, balanceAfter, reason(enum),
              refType, refId, unitCost?, batchNo?, expiryAt?, actorAccountId, createdAt
              -- append-only, never updated, never deleted
```

- Admin is simply the stock owner whose account role is ADMIN. No special case.
- Admin `sellPrice` is wholesale; `minimumRetailPrice` and `customerPrice` control customer
  retail. A distributor may price only inside that range and never above MRP.
- `reserved` is what v1 lacked: stock is reserved at order placement and released on
  cancellation/expiry, so two customers can't buy the last unit.
- `StockLedger.reason`: `PURCHASE_IN | SALE_OUT | TRANSFER_IN | TRANSFER_OUT | RETURN_IN |
  ADJUSTMENT | DAMAGE | RESERVE | RELEASE`.
- Every stock mutation goes through `inventory.service.applyDelta()` — the only writer.
  `onHand` is never `UPDATE`d anywhere else.
- A distributor buying from admin is one transaction producing `TRANSFER_OUT` on admin's
  StockItem and `TRANSFER_IN` on the distributor's. That relationship was invisible in v1.

---

## 5. Orders

```
Order       id, orderNo (human, sequence-backed),
            buyerAccountId, sellerAccountId,          -- explicit, no *Type columns
            channel(WEB|APP|DISTRIBUTOR_ASSISTED),
            status, paymentStatus, fulfilmentStatus,
            subtotal, discountTotal, taxTotal, shippingTotal, grandTotal, currency,
            shippingAddress JSONB,   -- SNAPSHOT, not a FK
            billingAddress  JSONB,
            checkoutQuoteId?, attributionSource, preferredDistributorAccountId?,
            quotedSellerAccountId?, fallbackApplied, fallbackReason?,
            placedAt, confirmedAt, shippedAt, deliveredAt, cancelledAt, cancelReason
OrderItem   id, orderId, variantId?,                  -- nullable, SetNull is legal here
            sku, productName, variantLabel,           -- SNAPSHOT (e.g. "500 ml · Red")
            unitPrice, quantity, discount, taxRatePct, taxAmount, lineTotal
OrderEvent  id, orderId, fromStatus, toStatus, actorAccountId, note, createdAt

CustomerCart      id, accountId (unique), createdAt, updatedAt
CustomerCartItem  id, cartId, variantId, quantity, unique(cartId, variantId)
CheckoutQuote     customerAccountId, cartId, sellerAccountId, preferredDistributorAccountId?,
                  attributionSource, status, fallbackReason?, totals, expiresAt, usedAt?
CheckoutQuoteItem quoteId, variantId, SKU/name/price/tax snapshots

Invoice      id, orderId (unique), invoiceNo (human, sequence-backed, "NX-INV-000123"),
             pdfUrl, pdfPublicId, issuedAt, createdAt
```

**Invoice (ADR-0016, added 2026-08-26; trigger corrected 2026-08-27).** One row per order, created
once — automatically, when `Order` transitions to `SHIPPED` — and never edited afterward, same
immutability reasoning as `OrderItem`. Trigger is shipment, not delivery: CGST Act Section 31 /
Rule 46 requires the tax invoice to issue at or before removal of goods for supply, i.e. dispatch
— generating it at `DELIVERED` (the original implementation) could be legally late if delivery
confirmation lags shipment by days. `invoiceNo` uses a Postgres sequence (`invoice_no_seq`), the
identical pattern `Order.orderNo` already established. The PDF itself always names
**NexaShopping** as seller of record — never the fulfilling distributor's `StockItem` —
regardless of who actually shipped the order; that's the whole point of ADR-0016. The row is
written in two steps: the number and row are reserved inside the same transaction that marks the
order shipped (cheap, pure DB), but the PDF is rendered and uploaded to Cloudinary *after* that
transaction commits — rendering + a network upload must never hold a database transaction open.

**Changes from v1**
- `userId + userType` / `vendorId + vendorType` → `buyerAccountId` / `sellerAccountId`.
  Both are real FKs to `Account`. `channel` records *how* it was placed.
- `addressId` FK → JSONB snapshot. An order must render identically in five years.
- `OrderItem` snapshots name/SKU/price, so renaming a product doesn't rewrite invoices.
  `variantId` becomes nullable specifically so `onDelete: SetNull` is legal — v1 declared
  SetNull on a required column, which cannot work.
- Full money breakdown instead of just `totalAmount` + `taxAmount`. Invoices and GST reporting
  need line-level tax.
- `OrderEvent` gives a real audit trail and replaces guessing from `updatedAt`.
- Status transitions are enforced by a state machine in `orders.service`, not by whatever
  string the client sent.
- A checkout quote locks customer-visible prices for ten minutes. A distributor is selected
  internally only if it can fulfil every cart line. If it cannot fulfil at quote time, or loses
  capacity before placement, the entire order uses primary-admin stock. No order is split and
  seller account identifiers are omitted from customer responses.

---

## 6. Payments

```
PaymentIntent  id, orderId? , creditRepaymentId?, accountId, amount, currency,
               provider, providerRef, status, idempotencyKey (unique),
               expiresAt, createdAt
PaymentEvent   id, paymentIntentId, providerEventId (unique), rawPayload JSONB,
               status, receivedAt        -- webhook dedupe lives here
Refund         id, paymentIntentId, amount, reason, status, providerRef
```

- v1 used self-referencing `parentTransactionId` for refunds. Refunds get their own table —
  clearer queries, no accidental recursion.
- `providerEventId` unique constraint is the idempotency guarantee: PhonePe can retry a
  callback ten times and the ledger moves once.
- Gateway payloads are stored raw for reconciliation and disputes.
- `PaymentIntent` can settle either an order or a credit repayment, hence two nullable FKs with
  a check constraint that exactly one is set.

---

## 7. Credit — ledger, not a balance column

```
CreditAccount  id, accountId (unique), creditLimit, currency,
               currentBalance,          -- maintained ONLY with a ledger insert
               status, openedAt
CreditEntry    id, creditAccountId, direction(DEBIT|CREDIT), amount, balanceAfter,
               reason(LIMIT_GRANT|LIMIT_REVOKE|ORDER_CHARGE|REPAYMENT|ADJUSTMENT|
                      INTEREST|WRITE_OFF|REVERSAL),
               refType, refId, actorAccountId, note, createdAt
               -- append-only
CreditRepayment id, creditAccountId, amount, paymentIntentId?, method, status,
                dueDate?, settledAt
```

**Changes from v1**
- `CreditAllocation` and `CreditTransaction` were two views of the same event with circular
  optional FKs. One `CreditEntry` table replaces both; an allocation is
  `reason = LIMIT_GRANT`.
- `availableCredit` as a directly-written column is gone. `currentBalance` is only ever written
  inside the same transaction as the `CreditEntry` insert, via
  `credit.service.post()` — the single writer. Available credit = `creditLimit - currentBalance`.
- A nightly job re-derives balances from the ledger and alerts on any mismatch. If they ever
  drift, we find out from a metric, not from a distributor.
- Row-level locking (`SELECT ... FOR UPDATE`) on the credit account during posting prevents
  double-spend from concurrent orders.
  > `[DECISION NEEDED]` Q4: interest/late fees? credit terms per distributor? partial
  > repayment allocation order (oldest-first)?

---

## 8. Supporting

```
KycSubmission   id, accountId, status, submittedAt, reviewedAt, reviewerAccountId,
                rejectionReason
KycDocument     id, submissionId, type(AADHAAR_FRONT|AADHAAR_BACK|PAN|GST|PASSBOOK|CHEQUE),
                fileUrl, extractedNumberEncrypted, verifiedAt
BankAccount     id, accountId, holderName, accountNumberEncrypted, ifsc, bankName, isPrimary
```
- v1 crammed bank details, Aadhaar, PAN, GST, and review state into one wide `KycDocument`
  row with one status. Resubmission overwrote history. v2 keeps submissions as attempts.
- **Aadhaar/PAN/account numbers are encrypted at rest** and never returned in list responses.
  Only the last 4 digits are exposed. This is a legal requirement, not a nice-to-have.

```
Cart / CartItem        status(ACTIVE|CONVERTED|ABANDONED), no fake unique hack.
                       Price is NOT stored on the item; carts re-price on read.
Banner                 placement enum, active window (startsAt/endsAt), targeting
Notification /
NotificationRecipient  template + per-recipient read state
Review                 one row, optional rating 1-5, verified-purchase flag,
                       moderation status. v1's split Rating/Review with a nullable
                       cross-link is collapsed.
AuditLog               actorAccountId, action, entityType, entityId, before/after JSONB
```
`priceAtAdd` on a cart item is removed deliberately: it created a stale-price bug surface.
Carts show live price; the price is frozen only at order placement.

---

## 9. Index checklist

Add with the migration that creates the table, not later:

```
Account(phone) unique · Account(email) unique · Account(role, status)
Product(categoryId, status) · Product(slug) unique · ProductVariant(sku) unique
StockItem(ownerAccountId, variantId) unique · StockItem(ownerAccountId, isListed)
StockLedger(stockItemId, createdAt desc)
Order(buyerAccountId, placedAt desc) · Order(sellerAccountId, status, placedAt desc)
Order(orderNo) unique · OrderItem(orderId)
PaymentIntent(idempotencyKey) unique · PaymentEvent(providerEventId) unique
CreditEntry(creditAccountId, createdAt desc)
```
Product search: Postgres `tsvector` + GIN index, or `pg_trgm` for fuzzy name matching. Do not
ship `contains` scans on a growing catalog.

---

## 10. Legacy data

Greenfield (ADR-0003). No `legacyId` columns, no compatibility shims — the v1 backup is not a
constraint on this schema.

If we later decide to bring anything across, it is a standalone script under
`nexa-server/scripts/import-v1/`, written once the schema is stable, run once, and deleted
afterwards. Realistic candidates in priority order: **categories and products** (worth keeping —
real data entry effort), **accounts** (only if distributors will be re-onboarded rather than
re-registered), **orders and credit history** (probably not worth it — the shapes differ enough
that a read-only archive of the old dump serves support better than a lossy import).

Before anything else in Phase 0: record where the v1 dump lives, verify it restores, and note
it in this file. A backup nobody has test-restored is not a backup.

### Backup verification (Task 0) — recorded 2026-08-13

**Source dump.** Produced by `pg_dump` of database `nexashopping` (see `backup.sh` in the v1
repo) → `/home/ubuntu/db_backup_<date>.sql` on the production VPS. A full copy of the v1
codebase (including `.env`, `docker-compose`, prisma schema) is retained locally at
`D:\Deploy\nexashopping`.

**Extract examined.** `db_backup_2025-09-27_extract.xlsx` (~208 KB, 213,366 bytes, 34 sheets,
one per table) — a human-readable export of the 2025-09-27 dump.

**Row counts (data rows, from the extract):**

| Table | Rows | | Table | Rows |
|---|---|---|---|---|
| Account | 31 | | CreditAccount | 21 |
| Product | 3 | | Distributor | 16 |
| Order | 31 | | User | 13 |

The extract confirms the v1 shapes this redesign targets: split `AdminInventory` /
`DistributorInventory` (+ two log tables), `CreditAllocation` + `CreditTransaction`, a
`Referral` table, and a `Token` table with a type discriminator.

**Still owed:** a true restore drill of the binary `.sql` dump into a throwaway Postgres
container (the xlsx is a derived extract, not restorable). Copy the `.sql` off the VPS to
object storage so the backup does not live only on one host — this is small data (31 orders),
so the realistic path is a one-off catalog/accounts import script later, not a full restore.
