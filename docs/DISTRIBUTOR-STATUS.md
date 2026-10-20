# Distributor App — Functionality Status

> **Dated snapshot:** This file describes the distributor app as of 2026-08-27. The diary submission feature was added later; see [the current overview](../README.md) and the [diary API contract](API.md#diary-cash-receipt-ledger-october-2026).

Snapshot as of 2026-08-27. Covers everything a distributor account touches: `nexa-server`
modules used by the distributor role, and every screen in `nexa-distributor`. Server routes were
read directly from each module's `*.routes.ts` — not inferred from docs. ✅ = built and wired
end-to-end (server + UI where applicable). ⬜ = not built. ⛔ = deliberately out of scope for
this version (see the linked ADR).

Update this file whenever distributor-facing scope changes — it drifts otherwise, same risk as
`API.md`.

---

## 1. Auth (login)

**Server** (`auth` module):
- ✅ `POST /auth/otp/request` — request a login code (Ping4SMS, distributor-only in this flow)
- ✅ `POST /auth/otp/verify` — verify code, issue session
- ✅ `GET /auth/me` — current session identity
- ✅ `POST /auth/logout`

**UI** (`nexa-distributor`):
- ✅ `login/page.tsx` — two-step phone → code form
- ✅ Session persisted, auth-gated redirect in `dashboard/layout.tsx`

---

## 2. Dashboard / overview

**Server:** ⬜ no dedicated endpoint — composed client-side from inventory, orders, customers,
and settlements reads.

**UI:**
- ✅ `dashboard/page.tsx` — welcome header, 4 KPI cards (inventory health, open orders,
  customers, payable balance), sales performance chart, inventory health donut, low-stock list,
  recent-activity feed, quick actions
- ✅ (2026-08-27) Fixed a real data-integrity bug: the inventory-health KPI and donut used to
  fall back to hardcoded placeholder numbers (`healthy || 92`, `low || 8`) whenever the real
  value was `0` — silently turning a genuine 0% health crisis into a fake "92%", or genuinely
  zero low-stock items into a fake "8 items low-stock". Fixed, plus a proper empty state ("No
  inventory yet") for a brand-new account. Same pass also replaced four other pieces of
  hardcoded/fake dashboard copy with real computed values: the "N payments need attention"
  banner (now counts real `PAYABLE` settlements, hides itself at zero), "Live sync · Xm ago"
  (now derived from the queries' actual `dataUpdatedAt`), the Customers KPI detail (now "N new
  this month" instead of a fabricated "+12.4%"), and the Payable-balance KPI detail (now "N
  payouts pending" instead of a fabricated "Due in 12 days"). Also deleted a literally-empty
  placeholder card that sat in the main grid rendering nothing.
- ✅ (2026-08-27) Redesigned the "Attention needed" (low-stock) and "Live feed" (recent
  activity) cards: finished a half-done purple→brand color migration, added severity coloring
  to the low-stock icon/badge, and fixed the activity feed's status dot — it used to be
  hardcoded by list *position* (2nd row always green regardless of that order's real status);
  now reuses the same status→tone map and `.orders-status` pill classes `orders-history.tsx`
  uses, so a status reads identically everywhere in the app.
- ⚠️ Note: sales chart still falls back to hardcoded placeholder values (`FALLBACK_SALES`) when
  fewer than 2 real orders exist — cosmetic-only, not a missing feature, but worth knowing
  before screenshotting this screen for anything real. Not fixed in the 08-27 pass; same
  category of issue, lower priority since a near-empty chart with real $0 bars is a more
  confusing initial impression than a placeholder trend line.

---

## 3. Shopping (buy from admin)

**Server** (`catalog` + `inventory` browse mode):
- ✅ `GET /categories`, `GET /products`, `GET /products/:id` (`catalog` module, public reads)
- ✅ `GET /inventory?sellerAccountId=<admin>` — browsing another account's listed stock
- ✅ `GET /inventory/:id` — exact catalog item detail
- ✅ `GET /accounts/admin` — resolves who "admin" is to buy from

**UI:**
- ✅ `dashboard/buy/page.tsx` — storefront grid, search, filters
- ✅ `dashboard/buy/[id]/page.tsx` — catalog item detail

---

## 4. Cart

**Server** (`cart` module):
- ✅ `GET /cart`
- ✅ `POST /cart/items`
- ✅ `PATCH /cart/items/:id`
- ✅ `DELETE /cart/items/:id`

**UI:**
- ✅ `dashboard/cart/page.tsx` — quantity +/-, remove, checkout

---

## 5. My Inventory & pricing

**Server** (`inventory` module):
- ✅ `GET /inventory` — own stock list
- ✅ `GET /inventory/:id` — exact item detail
- ✅ `POST /inventory` — receive stock (create-if-missing)
- ✅ `PATCH /inventory/:id` — pricing/listing update
- ✅ `GET /inventory/:id/retail-range`
- ✅ `POST /inventory/:id/adjust` — manual stock correction
- ✅ `GET /inventory/:id/ledger`
- ✅ (2026-08-27) `GET /inventory/export` — point-in-time stock snapshot, CSV or PDF
  (`format=csv|pdf`), no internal record IDs in either format
- ✅ (2026-08-27) `GET /inventory/ledger/export` — movement ledger across every stock item the
  caller holds, date-ranged, CSV or PDF
- ⬜ `POST /inventory/transfer` — admin-only (not a distributor capability by design, not a gap)

**UI:**
- ✅ `dashboard/inventory/page.tsx` — list, search, low-stock filter
- ✅ `dashboard/inventory/[id]/page.tsx` — detail: pricing, listing toggle, retail range, stock
  adjustment, ledger
- ✅ (2026-08-27) Snapshot export (date-independent) and ledger export (date range) buttons,
  each with a CSV/PDF format picker

---

## 6. Orders (my purchases from admin)

**Server** (`orders` + `payments` + `invoices` modules):
- ✅ `POST /orders` — place order (incl. `DISTRIBUTOR_ASSISTED` channel)
- ✅ `GET /orders` — list, scoped by role (buyer/seller)
- ✅ `GET /orders/:id`
- ✅ `GET /orders/:id/payment-status`
- ✅ `POST /orders/:id/confirm`, `/ship`, `/deliver` — fulfilment state machine
- ✅ PhonePe payment initiation + callback (`payments` module)
- ✅ (2026-08-27) GST invoice PDF: `invoices` module (no own router — reserved/generated inside
  `orders.service.shipOrder`, served via `GET /orders/:id/invoice`). Trigger point is
  **shipment**, not delivery — corrected mid-build to match CGST Act Section 31/Rule 46 (tax
  invoice must issue at or before removal of goods for supply). Company letterhead (legal name,
  GSTIN, address, contact) pulled from server config; sequential numbering via a Postgres
  sequence (`NX-INV-000123`); PDF rendered with `pdfkit`, stored on Cloudinary.
- ✅ (2026-08-27) `GET /orders/export` — GST sales register, one row per **invoiced** order,
  date-ranged by the invoice's `issuedAt` (not `placedAt`), CSV or PDF (`format=csv|pdf`)

**UI:**
- ✅ `dashboard/orders/page.tsx`, `dashboard/orders/[id]/page.tsx`
- ✅ `features/orders/payment-actions.tsx` — PhonePe pay/retry actions
- ✅ `distributor/payment/response/page.tsx` — PhonePe redirect landing page
- ✅ (2026-08-27) `features/orders/invoice-download.tsx` — invoice PDF download button, wired
  into both the purchase-order detail view and the sales-order detail view (§8), enabled once
  an order reaches `SHIPPED`/`DELIVERED`
- ✅ (2026-08-27) Sales-register export (date range + CSV/PDF picker) on the Sales page
  (`features/orders/sales-analytics.tsx`)
- ✅ (2026-08-27) Fixed `order-detail-page` (`dashboard/orders/[id]/page.tsx`) rendering
  narrower than the viewport on mobile (right-side gap) — same over-constrained CSS box model
  as `assisted-sale-page` (§7): `width: 100%` plus explicit `margin-left`/`margin-right: -24px`
  set together, which the spec resolves by silently discarding the coded right margin. Fixed
  with an explicit `width: auto`.

---

## 7. My Customers + assisted sale

**Server** (`customers` module, reuses `cart`/`orders`):
- ✅ `POST /customers` — attach/create by phone (idempotent)
- ✅ `GET /customers` — paginated list, search, status filter
- ✅ `GET /customers/:id`
- ✅ `PATCH /customers/:id` — private displayName/notes/status

**UI:**
- ✅ `dashboard/customers/page.tsx` — list, search, status filter, add/attach form
- ✅ `dashboard/customers/[id]/page.tsx` — detail, private notes editor
- ✅ `dashboard/customers/[id]/sale/page.tsx` — full assisted-sale flow: browse own inventory →
  customer-scoped cart → address checkout → place order
- ✅ (2026-08-27) Fixed a real mobile-only bug on this screen: the product-search toolbar had
  four overlapping patch CSS files (`assisted-sale-controls.css`, `-toolbar-fix.css`,
  `-alignment.css`, `-alignment-final.css`) fighting each other across three different, mismatched
  breakpoints (639/760/761px) with stacked `!important`s — the net effect was the page rendering
  narrower than the viewport on mobile (visible gap on the right). Consolidated into one clean
  mobile-first file (`assisted-sale-toolbar.css`). Separately, the page itself had `width: 100%`
  plus explicit `margin-left`/`margin-right: -24px` set simultaneously on mobile — an
  over-constrained CSS box model the spec resolves by silently discarding the *right* margin;
  fixed with an explicit `width: auto`.
- ✅ (2026-08-27) Fixed `customer-detail-modern` mobile padding inconsistency (same root cause
  and fix as `.profile-overview`/`.settlements-modern` — §11/§10) and gave it its own explicit
  `max-width: 1440px; margin: 0 auto` instead of relying on a Tailwind class on its page wrapper
  fighting an equal-specificity CSS rule; the wrapper `<div>` was then redundant and removed
  entirely (the page now renders `<CustomerDetailModern>` directly, same pattern as Settlements).
- 🔲 **Stale, not resumed:** a redesigned "Customers" list screen was attempted in Stitch
  (project `611146010874628502`) for visual polish back on 2026-08-26 — functionality above was
  already complete then and still is; this was a UI refresh pass only. Two generation attempts
  timed out and it was never revisited. No further action taken or currently planned; raise it
  again explicitly if the visual refresh is still wanted.

---

## 8. Sales (fulfilling my customers' orders)

**Server:** reuses `orders` module (`GET /orders?role=seller`, same confirm/ship/deliver/cancel).

**UI:**
- ✅ `dashboard/sales/page.tsx` — seller-side order list
- ✅ `dashboard/sales/[id]/page.tsx` (`features/orders/sales-order-detail.tsx`) — includes the
  same invoice-download button as the purchase-order view (§6); the distributor is the seller
  of record for these orders operationally, so they need the same PDF to hand to their customer

---

## 9. Trade credit & repayment

**Server** (`credit` module):
- ✅ `GET /credit/:accountId` — summary (limit, balance, available, overdue flag)
- ✅ `GET /credit/:accountId/charges`
- ✅ `GET /credit/:accountId/ledger`
- ✅ `GET /credit/:accountId/repayments`
- ✅ `POST /credit/:accountId/repayments` (start a PhonePe repayment)
- ✅ (2026-08-27) `GET /credit/:accountId/ledger/export` — statement export ordered
  **oldest-first** (unlike the UI's newest-first list, so it reads as a running balance like a
  bank statement), date-ranged, CSV or PDF. No internal record IDs in either format.
- ⬜ `PATCH /credit/:accountId` — admin-only limit changes (not a distributor capability, not a
  gap)

**UI:**
- ✅ `dashboard/credit/page.tsx` — balance metrics, outstanding charges, repayment history,
  ledger, "pay now" via PhonePe
- ✅ (2026-08-27) Ledger export (date range + CSV/PDF picker) in the ledger section header
- ✅ (2026-08-27) Fixed a real mobile bug on the metric cards: the icon badge was being pushed
  outside the card boundary on narrow screens (classic flexbox min-content overflow — the text
  block had no `min-width: 0`, so it refused to shrink and forced the fixed-size icon out past
  the edge). Also forces "Credit limit" onto two lines on mobile so all four metric cards read
  at a consistent height.

---

## 10. Settlements (payouts from admin)

**Server** (`settlements` module):
- ✅ `GET /settlements` — my payables
- ✅ `GET /settlements/payouts` — my payout history
- ✅ (2026-08-27) `GET /settlements/export` — payables statement, date-ranged on `createdAt`,
  oldest-first, CSV or PDF. Columns: Order No, Amount, Status, Eligible Date, Paid Date, Payout
  Reference, Method — no internal record IDs.
- ⬜ `POST /settlements/payouts` — admin-only (records a payout to a distributor, not a
  distributor capability)

**UI:**
- ✅ `dashboard/settlements/page.tsx` — payables list + payout history
- ✅ (2026-08-27) Export (date range + CSV/PDF picker) on the Payables section
- ✅ (2026-08-27) Fixed mobile padding inconsistency: `.settlements-modern` (like
  `.profile-overview` and `.customer-detail-modern`) had two or three competing horizontal-padding
  rules layered on top of each other — one adding `24px→16px`, another zeroing it with
  `!important` — landing on a different actual value depending on cascade order. Now relies
  solely on the dashboard shell's padding, matching Inventory/Customers/Credit's already-correct
  convention. Also fixed the export widget (date pickers + format picker + button) overflowing
  on mobile — needed `flex-wrap` and `min-width: 0` on its children, same fix applied
  pre-emptively to Credit's identical widget.

---

## 11. Profile

**Server** (`accounts` module):
- ✅ `GET /accounts/:id`, `PATCH /accounts/:id` (self: name/avatarUrl only)

**UI:**
- ✅ `dashboard/profile/page.tsx`
- ⚠️ Note: `dashboard/profile/[section]/page.tsx` (`business`/`security`/`notifications`) exists
  as pure static placeholder content — no real data, no working links inside it, and the
  `notifications` variant would dangle a promise for a feature that's explicitly out of scope
  (ADR-0012). It's already unreachable from normal navigation (`.profile-overview__links` is
  `display: none` in `globals.css`), so not currently a visible bug — just dead code worth
  deleting or wiring up properly, not doing both halfway.

---

## 12. Cross-cutting UI infrastructure

- ✅ Toast/feedback provider (`components/feedback.tsx`)
- ✅ Loading/empty/error states on every list screen (per `nexa-distributor/CLAUDE.md` rule)
- ✅ Mobile navigation drawer (Vaul), responsive layout throughout
- ✅ (2026-08-27) CSV/PDF export as a first-class pattern: every export screen (Sales, Credit,
  Inventory ×2, Settlements) gets a From/To date picker (where date-ranged) plus a CSV/PDF
  format select and an Export button, all calling `api.download()` (added to `lib/api.ts`
  alongside `api.get`). No export anywhere includes a raw internal record ID.
- ✅ (2026-08-27) Fixed `.assisted-sale-page` (§7) / `.order-detail-page` (§6) rendering narrower
  than the viewport on mobile — an over-constrained CSS box model, not a layout-intent bug;
  same root cause could recur anywhere `width: 100%` and both explicit `-Npx` margins are set
  together, worth grepping for if a similar "gap on the right" report comes in on another screen.
- ⛔ **Global header search** — was built (debounced, routed to per-section search), **removed
  per explicit instruction on 2026-08-26**. Per-page local search bars (customers, inventory,
  buy) are untouched and remain the only search surfaces.

---

## Not built — real gaps or explicit scope cuts

| Item | Status | Reference |
|---|---|---|
| **KYC submission** | ⛔ Dropped from this version entirely. | ADR-0020 |
| **Notifications** (any: in-app, FCM, SMS) | ⛔ Deferred until v1 parity. | ADR-0012 |
| **Refunds** (distributor-visible or otherwise) | ⛔ Deferred, handled manually for now. | ADR-0019 |
| **Global header search** | ⛔ Removed by request. | This doc, §12 |
| **v1 API compatibility layer** | ⬜ Not scoped — needed only when the still-live v1 app is migrated. Blocked on getting v1's actual route shapes. | ADR-0017 |
| **Profile settings sub-pages** (business/security/notifications) | ⬜ Static placeholder only, already hidden from nav. Low priority — delete or build for real. | This doc, §11 |
| **Customers screen visual refresh** | ⬜ Stale — two Stitch attempts timed out 2026-08-26, never resumed. Functionality unaffected. | This doc, §7 |

---

## Scorecard

- **Server:** every distributor-relevant endpoint across `auth`, `accounts`, `catalog`,
  `inventory`, `cart`, `orders`, `payments`, `customers`, `credit`, `settlements`, and the new
  `invoices` module is implemented. Billing/invoicing — the last real gap as of 2026-08-26 — is
  now built end-to-end (PDF generation, sequential numbering, download UI) as of 2026-08-27,
  along with CSV/PDF exports across all four money-relevant screens.
- **UI:** all 11 functional areas have a complete screen set (list/detail/create/mutate states,
  loading/empty/error handled). The 2026-08-27 pass additionally: fixed several mobile-only
  rendering bugs across Customer Detail, Assisted Sale, Order Detail, Settlements, and Credit
  (a recurring pattern, not unrelated one-offs — an over-constrained CSS box model in three
  places, competing padding-inline rules in three places, and the same flexbox min-content
  overflow in three separate metric-card components), replaced five pieces of
  fabricated/hardcoded dashboard data with real computed values, and redesigned two dashboard
  cards to finish a half-done color migration.
- **Real gaps:** the v1 compat layer (blocked on external input, not actionable yet) and the
  Customers screen visual refresh (stale, not resumed — flag if still wanted). Everything else
  outstanding is either a deliberate scope cut (KYC, notifications, refunds — all ADR'd) or
  low-priority dead code (the hidden profile sub-pages).
