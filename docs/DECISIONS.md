# DECISIONS (ADR log)

Every non-obvious choice gets an entry, newest at the bottom. Keep it short — context,
decision, consequence. When an answer from `OPEN-QUESTIONS.md` is settled, it becomes an entry
here and is deleted from there.

ADRs are **not edited after the fact**. When a decision changes, the old entry keeps its
original wording and gains a `superseded by` note — the reasoning at the time is the useful
part. Only the status line changes.

Template:
```
## ADR-NNNN — <title>
Date: YYYY-MM-DD · Status: Accepted | Superseded by ADR-XXXX
Context:      what forced a choice
Decision:     what we chose
Consequences: what this makes easy, what it makes hard, what we accept
```

---

## ADR-0001 — Rebuild rather than refactor v1
Date: 2026-08-13 · Status: Accepted; repo layout superseded by ADR-0006

**Context.** v1 works but its structure blocks change: routes and controllers are triplicated
per role, inventory is modelled twice (`AdminInventory` / `DistributorInventory` plus two log
tables), orders join to mutable rows instead of snapshotting, money mixes `Float` and
`Decimal`, credit is a mutable balance column with no enforced ledger, and several
`onDelete: SetNull` rules are declared on non-nullable columns. `authController.ts` alone is
2,125 lines. Fixing these in place would touch nearly every file anyway.

**Decision.** Rebuild in a pnpm/Turborepo monorepo with a layered API
(routes → controller → service → repository → policy), one shared contract package, and a
redesigned schema (see `docs/DATA-MODEL.md`). The server is dockerized; the three Next.js
surfaces deploy to Vercel.

**Consequences.** Higher upfront cost and a migration or cutover decision (Q1). In exchange:
one code path per capability, a schema that can carry variants, reservations, and a real
credit ledger, and shared types that make server/panel drift a compile error.

---

## ADR-0002 — Role-based routing replaced by a permission layer
Date: 2026-08-13 · Status: Accepted

**Context.** v1 exposed `/admin/api/v1/*`, `/distributor/api/v1/*`, and `/api/v1/*` with a
separate route file and often a separate controller branch for each — the single largest source
of duplicated code.

**Decision.** One route per resource under `/api/v1`. Coarse checks via
`authorize('order:read:any')`; row-level checks via a per-module `*.policy.ts` called from the
service. Result scoping is derived from `req.ctx.actor`.

**Consequences.** Permission tests become critical — a missing check is now a real
vulnerability rather than being accidentally prevented by the URL. We therefore require a
permission-matrix test covering every role × every endpoint before Phase 1 exits.

---

## ADR-0003 — Greenfield database; v1 data is an optional one-off import
Date: 2026-08-13 · Status: Accepted (resolves Q1)

**Context.** v1 services are shut down, nothing is serving production traffic, and a database
backup has been taken. There is no live data to preserve continuity for.

**Decision.** v2 designs its schema without any compatibility constraint from v1. No `legacyId`
columns, no dual-write, no parallel run. If any historical data is wanted later (catalog,
accounts), it is handled as a standalone import script written *after* the schema is stable,
reading the backup — never as a schema constraint.

**Consequences.** Phase 0 is unblocked and every redesign in `docs/DATA-MODEL.md` can be
implemented as designed. The cost is that old order/credit history does not carry over unless
we explicitly choose to import it; the backup must be retained and its location documented
before we touch anything.

---

## ADR-0004 — Next.js App Router for all three web surfaces
Date: 2026-08-13 · Status: Accepted (resolves Q2); repo layout superseded by ADR-0006

**Context.** v1 panels were Vite SPAs mounted at `/admin` and `/distributor`, each with its own
Redux store containing a slice per feature — the same fourteen slices duplicated across two
apps.

**Decision.** Admin, distributor, and the marketing site are all Next.js App Router apps
deployed as three Vercel projects from the monorepo. Server state is TanStack Query only;
Redux/Zustand is reserved for genuine client state. `packages/ui` is shared by all three.
Panels move to subdomains (`admin.` / `distributor.`) so session cookies are scoped by
`Domain=.nexashopping.in` rather than by path.

**Consequences.** Existing panel screens are rewritten rather than ported — but they were going
to be rewritten anyway, since the API contract changes shape. One framework across all web
surfaces means one shared component library and one auth pattern. DNS for the two subdomains
must be provisioned in Phase 0.

---

## ADR-0005 — Product variants are first-class; inventory keys on variant
Date: 2026-08-13 · Status: Accepted (resolves Q3)

**Context.** v1 had one price per product, no SKU, and EAV attributes — so pack sizes and
colours could only be modelled as separate products, and stock could not be tracked per
sellable unit.

**Decision.** `ProductVariant` is the sellable unit and carries the SKU. Every product has at
least one variant (a single-variant product is the degenerate case, created automatically so
there is no second code path). Inventory, cart items, and order items all reference
`variantId`, never `productId`. Attributes are split into variant axes (`isVariantAxis = true`,
e.g. size) and descriptive attributes.

**Consequences.** Catalog CRUD and the admin product form are meaningfully more complex —
variant matrix generation, per-variant pricing and stock. In exchange, stock is accurate at the
unit actually sold, and adding a pack size later is data entry rather than a migration. Every
query path must remember that "product stock" is an aggregate over variants.


---

## ADR-0006 — Multi-repo under a GitHub org, shared code as published packages
Date: 2026-08-13 · Status: **Superseded by ADR-0007** (the multi-repo part stands; the shared
packages do not)

**Context.** The org will host separate repos for server, admin, distributor, website, app, and
infra. This gives independent deploys, per-repo access control, and clean Vercel wiring — but
removes the ability to import shared code by path, which was the mechanism ADR-0001 relied on
to stop server and panels drifting apart.

**Decision.** Nine repos (`nexa-docs`, `nexa-shared`, `nexa-ui`, `nexa-server`, `nexa-admin`,
`nexa-distributor`, `nexa-website`, `nexa-app`, `nexa-infra`). Contracts, SDK, and the `Money`
helper live in `@nexashopping/shared`, published to GitHub Packages with changesets and pinned
at an exact version by every consumer. `nexa-docs` holds this instruction set; other repos carry
a short `CLAUDE.md` pointing at it.

**Consequences.** A contract change becomes three PRs across three repos in a fixed order
(shared → server → panels) instead of one commit. That friction is the price of the split, and
the rule that makes it survivable is: **never hand-write a local type to work around a missing
contract.** The API must stay backward-compatible within a major so panels can deploy on their
own schedule. Cross-repo CI is also weaker — nothing automatically proves the server and a panel
agree, so the pinned-version discipline is doing real load-bearing work.


---

## ADR-0007 — No shared packages; `API.md` is the contract; npm
Date: 2026-08-13 · Status: Accepted (supersedes the packaging half of ADR-0006)

**Context.** ADR-0006 proposed `@nexashopping/shared` and `@nexashopping/ui` published to
GitHub Packages, to stop the server and panels drifting apart. In practice that requires a
registry, an access token in every CI and Vercel project, peer-dependency care, Tailwind
content globs reaching into `node_modules`, and three coordinated PRs for a single field
rename. The team is one developer working with Claude Code. The machinery costs more than the
drift it prevents.

**Decision.** Five independent repos, npm, no shared packages, no workspaces. `nexa-docs/API.md`
is the contract: the server's Zod schemas and each panel's `src/lib/types.ts` are both written
from it, by hand. Panels each own their components; admin and distributor may hold near-
identical copies. Deploying a panel is: import the repo into Vercel, set `NEXT_PUBLIC_API_URL`,
deploy.

**Consequences.** Nothing mechanically catches API drift, so `API.md` must be updated in the
same session as the server change — this is now the single most load-bearing discipline in the
project. Renaming or removing a response field becomes a manual grep across panel repos;
adding fields stays safe, so prefer adding. UI components will be duplicated between admin and
distributor, which is accepted. The risk is bounded by one existing rule: **panels never compute
money**, so any drift is a display bug, never an arithmetic one.

**Revisit if** more than one or two silent-drift incidents occur, or a second developer joins.
At that point introduce a shared contract package — as a new ADR, not pre-emptively.

---

## ADR-0008 — Fixed demo OTP for distributor login, ahead of DLT verification
Date: 2026-08-15 · Status: Superseded by ADR-0009

**Context.** Distributor login is phone OTP, but real SMS delivery needs a DLT (TRAI) sender
registration that isn't done yet — the same blocker noted against `POST /auth/otp/*` since
Phase 1. The distributor panel otherwise has nothing to build against; waiting for DLT blocks
all distributor-panel work, not just login.

**Decision.** `OTP_DEMO_MODE` (env, off by default) makes `POST /auth/otp/verify` accept one
fixed code (`OTP_DEMO_CODE`, default `123456`) for **any** phone number that already belongs to
an active `DISTRIBUTOR` account. `POST /auth/otp/request` never reveals whether a phone is
registered (always returns `{ ok: true }`) so the interface shape matches what a real SMS
provider will need later — swapping in real delivery touches only `auth.service.ts`'s two OTP
functions. Distributor accounts still only come from admin onboarding
(`POST /accounts/distributors`); OTP never creates one.

**Consequences.** While `OTP_DEMO_MODE=true`, anyone who knows a distributor's registered phone
number can sign in as them — there is no real second factor. This is acceptable only because
distributor accounts are few, admin-onboarded, and known; it is not acceptable once the panel
or the account base is real/public. **This must be turned off (or replaced with real SMS
delivery) before any actual distributor onboarding beyond internal testing**, and before the
mobile app (customer OTP) reuses the same pattern. Tracked informally here and in `API.md`'s
Auth section until DLT is verified and this ADR can be closed out.

---

## ADR-0009 — Ping4SMS DLT OTP with PostgreSQL challenges
Date: 2026-08-16 · Status: Accepted (supersedes ADR-0008)

**Context.** The Ping4SMS DLT sender and login template are verified. Distributor login can
replace the fixed demo code, but Redis remains intentionally deferred until the payments and
background-job phases.

**Decision.** Send distributor login codes through Ping4SMS transactional route 2 using the
approved sender, template ID, and exact DLT message. Store one scrypt-hashed challenge per
distributor account in PostgreSQL. Codes expire after five minutes, requests have a 60-second
resend cooldown, verification allows five failed attempts, and a successful code is atomically
deleted before a session is created. The request endpoint keeps returning `{ ok: true }` for
unknown or ineligible numbers to prevent account enumeration.

**Consequences.** Distributor login no longer has a demo bypass. Ping4SMS credentials become
required server configuration. PostgreSQL handles this low-volume authentication state without
adding Redis solely for OTPs; Redis can still be introduced later for distributed rate limits,
payments, or reliable background jobs. Customer signup/login remains a separate future flow.

---

## ADR-0010 — PostgreSQL fixed-window limits for distributor OTP
Date: 2026-08-16 · Status: Accepted

**Context.** The real distributor OTP flow needs protection against SMS-cost abuse and repeated
verification attempts. Redis is still deferred, and the API is reached through one Caddy proxy
hop in the current deployment.

**Decision.** Store fixed-window counters in PostgreSQL, keyed by action, a SHA-256 subject hash,
and window duration. Limit OTP requests to 20 per hour per client IP, accepted SMS sends to five
per hour and ten per day per phone, and verification requests to 30 per hour per client IP.
Phone quotas are consumed transactionally before reserving a challenge and are refunded by exact
lease when cooldown or provider failure prevents a send. All request-endpoint denials retain the
generic `{ ok: true }` response. Express trusts exactly one proxy hop so the rate limiter sees
Caddy's forwarded client IP.

**Consequences.** Concurrent API instances share atomic limits without another infrastructure
service, and the table holds at most one counter row per action/subject/window combination.
Raw phone numbers and IP addresses are not stored in rate-limit rows or logs. The API must remain
private behind Caddy; changing the proxy topology requires reviewing the trust setting. Redis can
replace this implementation later if traffic or operational needs justify it. A daily maintenance
command removes buckets that have been expired for more than 24 hours.

---

## ADR-0011 — Cloudinary for product media (resolves Q11)
Date: 2026-08-21 · Status: Accepted

**Context.** Product media needed a real storage backend rather than a plain URL string.

**Decision.** Use Cloudinary for product media, with signed uploads. Implemented in
`nexa-server/src/integrations/cloudinary.ts` and `src/modules/catalog/product-media.service.ts`,
migration `20260821100000_add_cloudinary_product_media`.

**Consequences.** No S3/R2 migration needed. Cloudinary credentials are required server
configuration (`src/core/config.ts`).

---

## ADR-0012 — Notifications and shipping/fulfilment integration deferred until v1 parity is reached
Date: 2026-08-26 · Status: Accepted

**Context.** Q8 (notifications) and Q10 (shipping/courier integration) were open questions with
no immediate pressure to resolve — v1 feature parity is the nearer goal.

**Decision.** Do not build notification delivery (FCM/SMS/WhatsApp/email) or a courier/aggregator
integration until v1 parity is reached. Manual dispatch and no in-app notifications remain the
status quo until then.

**Consequences.** `Notification` model / routes (if any) stay unbuilt or unused; shipping stays a
bare reference string. Revisit both once v1 parity is declared done — do not build ahead of that
milestone without asking first.

---

## ADR-0013 — Credit terms (resolves Q4)
Date: 2026-08-26 · Status: Accepted

**Context.** Q4 blocked Phase 6. The `credit` module was already implemented in
`nexa-server/src/modules/credit/` ahead of formal sign-off on these terms; this ADR confirms the
built behaviour matches intent and records the two points that weren't yet observable in code.

**Decision.**
1. Credit is interest-free, no late fee.
2. Fixed limit per distributor, set by admin (`creditLimit` on the credit account) — not scaled
   by a computed credit score.
3. Repayment is on demand, no fixed cycle/due-date terms.
4. Partial repayment allocates oldest charge first (`credit.repository.ts` orders outstanding
   charges by `dueAt asc, createdAt asc, id asc`).
5. An overdue distributor is blocked from placing further credit orders until the overdue charge
   is repaid (`credit.service.ts` `hasOverdueChargeTx` check) — not just flagged.
6. Distributors only; end customers do not get credit (no `CUSTOMER` role path in the credit
   module).

**Consequences.** Q4 is fully resolved; Phase 6 is unblocked in the roadmap sense (the module
already exists and matches this ADR). No interest-accrual logic is needed. If interest or cyclical
terms are wanted later, that's a new ADR, not a retrofit assumption.

---

## ADR-0014 — Referral is attribution-only, distributor-refers-distributor (resolves Q6)
Date: 2026-08-26 · Status: Accepted

**Context.** `DistributorProfile.referredByAccountId` records which distributor referred another
distributor onto the platform. This is unrelated to `DistributorCustomer` /
`customerProfile.acquisitionSource` / `acquiredByAccountId`, which track which distributor
acquired an end customer — that customer-acquisition tracking already exists and is untouched by
this ADR.

**Decision.** Referral (`referredByAccountId`) is attribution-only: recorded for
reporting/relationship purposes, no reward, no payout, no ledger entry. Scope stays
distributor-refers-distributor, matching the current schema — not extended to customer referral.

**Consequences.** No referral-reward feature is built. If a referral incentive is wanted later,
that's a new ADR with its own ledger design, not a retrofit onto this field.

---

## ADR-0015 — Host crontab + scripts for background jobs; no BullMQ/Redis worker (resolves Q7)
Date: 2026-08-26 · Status: Accepted

**Context.** Two scheduled jobs already run successfully this way: nightly Postgres backup and
daily auth rate-limit-bucket cleanup (`DEPLOY.md` §6–7), each a host `crontab` entry running
`docker compose exec -T api npm run <script>`. `scripts/expire-orders.ts` exists but, as of
2026-08-26, is not yet wired into crontab — nothing currently expires unpaid orders or stale
PhonePe payment intents on a schedule.

**Decision.** Keep the crontab + script pattern for scheduled jobs going forward (order expiry,
stale payment-intent cleanup, and future jobs like notifications) rather than introducing a
BullMQ worker. This also means Redis stays deferred for this reason specifically — it may still
come back later for other reasons (see `nexa-server/CLAUDE.md`).

**Consequences.** No retry/dead-letter machinery exists for any of these jobs — a run that fails
silently just waits for the next cron tick. Which jobs need stronger reliability guarantees
(retries, alerting) than "run again next cycle" is intentionally left undecided per-job rather
than answered uniformly here; revisit if a specific job (e.g. payment reconciliation) turns out
to need it. `scripts/expire-orders.ts` still needs a crontab entry added — that's an open
implementation gap, not a design question.

---

## ADR-0016 — GST invoicing: required at launch, flat tax rate, NexaShopping as seller of record (resolves Q9)
Date: 2026-08-26 · Status: Accepted

**Context.** Orders were built against Q9 being unanswered: flat `taxRatePct` per line (no HSN,
no CGST/SGST/IGST split), and seller-of-record left implicit as "whoever's `StockItem` the order
draws from" (`prisma/schema.prisma` comment above the Orders section). No invoice numbering or
invoice model exists yet anywhere in the codebase — this ADR is a design decision that still
needs implementation, not a description of what's built.

**Decision.**
1. A GST-compliant invoice, with sequential invoice numbering, is required at launch — not
   deferred.
2. Tax stays a flat percentage per line item (`taxRatePct`, as already modelled). No HSN codes,
   no state-aware CGST/SGST/IGST split.
3. **NexaShopping (the platform) is the seller of record on every order, regardless of whether
   admin or a distributor's `StockItem` fulfilled it.** This changes the implicit behaviour noted
   in the schema comment above (which read seller-of-record off the fulfilling `StockItem`) —
   invoicing must use one platform-wide GST registration and one invoice number series, not a
   per-distributor series.

**Consequences.** Needs actual implementation: an invoice model/table with a sequential number
generator (likely a Postgres sequence, matching the existing `orderNo` pattern), invoice
generation on order completion, and invoice numbering scoped to the platform rather than per
seller. The schema comment at `prisma/schema.prisma` (Orders section) and
`nexa-docs/docs/API.md`'s Cart & Orders status note should be updated to point here instead of
describing Q9 as unanswered. Since the platform is always seller of record, distributor GST
registration status becomes irrelevant to invoicing — only NexaShopping's own GST details matter.

---

## ADR-0020 — KYC submission dropped from this version's scope (Phase 7/8)
Date: 2026-08-26 · Status: Accepted

**Context.** ROADMAP.md's Phase 7 (admin) and Phase 8 (distributor) exit criteria both listed KYC
("KYC review" / "KYC submission") as required. Nothing has been built for it anywhere — no schema
field, no route, no screen — and everything else on both phases' lists is otherwise done (see the
2026-08-26 distributor functionality audit).

**Decision.** Drop KYC submission/review from this version entirely, alongside notifications
(already deferred per ADR-0012 — same "not until v1 parity" posture applies here too). Not a
partial build, not a stub — out of scope for now.

**Consequences.** Phase 7 and Phase 8 exit criteria in `ROADMAP.md` should be edited to remove
the KYC line item so those phases can be considered met without it. Distributor onboarding stays
admin-driven with no compliance-document collection step. Revisit if/when the platform actually
needs KYC (e.g. a payment gateway or regulatory requirement forces it) — treat that as a new
scope addition, not a resumption of a half-built feature, since nothing was started. — v1 mobile app stays live; server needs a v1 compatibility layer (resolves Q12)
Date: 2026-08-26 · Status: Accepted, compatibility layer NOT YET SCOPED

**Context.** A customer-mobile API foundation already shipped on v2 (phone OTP, cart with price
lock, orders) — see ROADMAP.md's "Customer mobile commerce" section. That, plus a
`seed-play-review-customer.ts` script and `nexa-website`'s `/delete-account` page (Play Store
data-deletion compliance), initially looked like evidence the app was already rebuilt against v2.
It is not: the existing store app is still v1 and must keep working during the v2 rollout.

**Decision.** The v1 mobile app keeps running against v1-shaped endpoints; the server needs a
compatibility layer so it keeps working while the app is separately rebuilt (by another
tool/developer, out of scope for this workspace — no `nexa-app` repo exists here and none needs
to be created here).

**Consequences — NOT YET ACTIONABLE.** This workspace has no v1 API source, route list, or
request/response shapes to design the shim against: `nexa-server`'s only "v1" is the *new*
`/api/v1` prefix on the rebuilt v2 schema (confirmed by search — no legacy routes exist); ADR-0003
recorded that v1 was shut down with only a **database** backup taken, and that backup isn't in
this workspace either. **Before this compat layer can be scoped or built, the v1 API's actual
endpoint shapes need to be supplied** (old repo, route docs, or Postman/OpenAPI export — wherever
that lives). Until then, treat v2 API changes as safe only insofar as they don't assume anything
about what the v1 app currently calls or expects back.

---

## ADR-0018 — Solo developer, no staging yet, no CI yet (resolves Q13)
Date: 2026-08-26 · Status: Accepted

**Context.** Phase 0's plan called for CI in every repo and a provisioned staging environment.
Neither exists: no `.github/workflows` in any of `nexa-admin`, `nexa-distributor`, `nexa-server`,
`nexa-website` (confirmed by search), and there's no separate staging VPS/DB — v1 is still the
only thing serving real production traffic (see ADR-0017), so v2 currently carries little
deploy risk by construction.

**Decision.** Solo developer (with Claude Code, occasionally ChatGPT Codex) — matches the
assumption ADR-0007 already made. No staging environment for now; test carefully against
production once close to cutover, revisit if that gets riskier. No CI for now; keep relying on
`npm run lint && npm run typecheck && npm test` run manually/by-agent before commits, as has been
the practice.

**Consequences.** ADR-0007's "revisit if a second developer joins" trigger is confirmed still
unmet. Since v2 isn't live yet, skipping staging is lower-risk than it would be post-cutover —
**revisit staging before actual cutover**, not after. Skipping CI means regressions are only
caught if the manual command sequence is actually run every time; there's no safety net if a
change ships without it.

---

## ADR-0019 — Payments: PhonePe (vendored), COD modeled, admin-side payouts to distributors, refunds deferred (resolves Q5)
Date: 2026-08-26 · Status: Accepted

**Context.** The `payments` and `settlements` modules already exist and mostly answer Q5; this
ADR confirms the built behaviour and records the two points that weren't yet observable in code.

**Decision.**
1. **Gateway:** PhonePe (`src/integrations/phonepe.ts`, `payments` module). Vendored locally as
   `"pg-sdk-node": "file:./src/libs/pg-sdk-node/package"` — same shape as v1's vendoring, but not
   an oversight: **no official PhonePe package is published to a registry usable from Docker**, so
   a local file dependency was the only option, not a rejected alternative. Kept as-is.
2. **COD:** modeled — `PaymentMethod` enum includes `COD` alongside `PHONEPE` and `CREDIT`.
3. **Settlement direction:** the platform (seller of record, per ADR-0016) holds customer payment.
   `DistributorPayable` tracks what's owed to a distributor per order (`HELD` → `eligibleAt` →
   `paidAt`); an admin manually records a `DistributorPayout` (one per distributor, allocated
   across payables via `DistributorPayoutAllocation`). Distributors are paid out by admin, not the
   reverse, and not automatically via gateway — it's an admin-recorded action.
4. **Refunds:** deferred. Not implemented (no refund logic in `payments.service.ts`, though the
   vendored SDK exposes refund types unused). Handle any refunds manually/offline for now.

**Consequences.** Q5 is resolved and Phase 5 is unblocked in the roadmap sense — the module
already exists and matches this ADR except refunds. Refund logic (full/partial, approval flow,
wiring to the SDK's refund calls) is explicitly known-missing, not silently absent — build it
when refunds are actually prioritized, not before.

---

## ADR-0021 — Diary is a cash-receipt ledger, not an installment scheduler
Date: 2026-10-20 · Status: Accepted (records the October 2026 implementation)

**Context.** Admins and distributors need to record actual customer cash handovers, while customers need a read-only history. The user-facing concept is called a diary, and the admin UI offers a “Start diary” shortcut.

**Decision.** One handover creates one `DiaryEntry` with an audit event. A distributor can submit only for an active linked customer; an admin confirms or reviews the entry. Approved, non-reversed entries alone contribute to the customer's total. Selecting a customer does not create an enrollment record, installment schedule, payment promise, or automatic charge. The first confirmed receipt starts the visible ledger. Reversals retain the original entry and reason. The separate Vite portal is read-only and uses existing-account OTP.

**Consequences.** The cash ledger stays separate from shopping payments and credit. A later request for contractual installment schedules or account enrollment needs an explicit product and legal decision, a new data model, and separate API work; it must not be inferred from the static monthly reference metadata.
