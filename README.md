# NexaShopping engineering reference

This repository holds cross-repository architecture notes, API contracts, and historical plans. **Start here.** The status below was checked against the local source repositories on 2026-10-25; it does not certify the state of production deployments or live email delivery.

## Active repositories

| Repository | Purpose | Deployment |
| --- | --- | --- |
| [nexa-server](https://github.com/NexaShopping/nexa-server) | Express API, Prisma schema and migrations | Docker Compose on the VPS (`api`, `postgres`, `caddy`) |
| [nexa-admin](https://github.com/NexaShopping/nexa-admin) | Admin panel | Vercel |
| [nexa-distributor](https://github.com/NexaShopping/nexa-distributor) | Distributor panel | Vercel |
| [nexa-website](https://github.com/NexaShopping/nexa-website) | Public website | Vercel |
| [nexa-dairy](https://github.com/Nexashopping-git/nexa-dairy) | Read-only diary portal; local folder `nexa-diary` | Cloudflare Worker static assets |
| [nexa-docs](https://github.com/Nexashopping-git/nexa-docs) | This reference | GitHub only |

The code and migrations in each application repository are authoritative for implemented behavior. The [API contract](docs/API.md) is a cross-repository reference, but older sections have not all been re-audited against the October 2026 implementation. The server's [customer mobile OpenAPI contract](https://github.com/NexaShopping/nexa-server/blob/main/docs/openapi/customer-mobile.yaml) covers that client specifically. Treat the August plans and schema in this repository as historical design unless current code confirms them.

## Diary feature

- Existing active customers and distributors sign in to the separate portal with existing-account OTP. The portal has no registration, payment, or entry-creation flow.
- Distributors submit cash handover entries only for active customers in their portfolio. Submissions remain pending until an admin reviews them.
- Admins search any active customer, record a confirmed cash receipt, and approve, reject, or reverse entries. The admin home has a **Start diary** shortcut; selecting a customer followed by the first confirmed receipt starts that customer's ledger. There is no separate enrollment or installment-schedule record.
- An approved, non-reversed entry contributes to the customer's total. The reference amount in the API is descriptive; it does not schedule or charge installments. See the [diary API section](docs/API.md#diary-cash-receipt-ledger-october-2026).

The user-facing hostname requested for this portal is `dairy.nexashopping.in`. Some earlier files say `diary.nexashopping.in`; use the actual deployed hostname consistently in Cloudflare and `CORS_ORIGINS`.

## Distributor sign-in

The distributor panel uses Firebase email/password or Google sign-in, then exchanges the verified Firebase ID token for a NexaShopping session at `POST /auth/distributor/login`. Admins assign each distributor's email. The server sends a welcome message from NexaShopping Gmail with a Firebase password link; the panel also offers **Reset password**. The former distributor panel OTP endpoints are retired; the separate diary portal still has its own existing-account OTP flow. See [the rollout guide](docs/DISTRIBUTOR-AUTH-ROLLOUT.md).

## Deployment pointers

- The API URL is `https://api.nexashopping.in`. Production Compose reads `nexa-server/.env` through `env_file`; after a CORS-only change, recreate the API container with `docker compose up -d --no-deps --force-recreate api` from the server directory.
- Admin and distributor panels use `NEXT_PUBLIC_API_URL=https://api.nexashopping.in` at build time. The Vite diary portal uses `VITE_API_URL=https://api.nexashopping.in` at build time.
- Cloudflare DNS is authoritative for `nexashopping.in`. Keep Vercel DNS records for the existing sites when editing the zone. Attach `dairy.nexashopping.in` as a Worker Custom Domain; the Worker serves `dist/` with a single-page-application fallback.
- The VPS runbook is [nexa-server/DEPLOY.md](https://github.com/NexaShopping/nexa-server/blob/main/DEPLOY.md). Diary rollout and audit notes are [DIARY-OPERATIONS.md](https://github.com/NexaShopping/nexa-server/blob/main/docs/DIARY-OPERATIONS.md).

## Production monitoring rollout (2026-10-26)

The server repository now has an optional Better Stack rollout in
[docs/monitoring.md](https://github.com/NexaShopping/nexa-server/blob/c71dc92/docs/monitoring.md):
safe request summaries, a bounded Vector syslog collector without a Docker
socket mount, host resource sampling, dashboard/alert setup, and rollback.
Repository checks do not prove that Better Stack ingestion or notifications are
active in production. Those require a Better Stack source, private VPS token,
operator setup, and a maintenance-window activation.

## Reference index

- [API.md](docs/API.md): endpoint shapes and diary contract
- [ARCHITECTURE.md](docs/ARCHITECTURE.md), [DATA-MODEL.md](docs/DATA-MODEL.md), [ROADMAP.md](docs/ROADMAP.md): original design and phased plan
- [DECISIONS.md](docs/DECISIONS.md): decisions and supersessions
- [DISTRIBUTOR-STATUS.md](docs/DISTRIBUTOR-STATUS.md): dated distributor feature snapshot

The local `.worktrees/nexa-distributor-payments` snapshot contains six source blobs that are absent from distributor Git history as of this check. Keep that snapshot until those differences are reviewed. It is not an active Git worktree.
