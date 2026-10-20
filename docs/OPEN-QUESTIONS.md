# OPEN QUESTIONS

Anything marked **BLOCKING** stops the phase it's attached to. Answer, then move it to
`docs/DECISIONS.md` as an ADR and delete it from here.

---

## Resolved

| Was | Answer | Recorded as |
|---|---|---|
| Q1 data strategy | Greenfield. v1 is shut down, backup taken; old data is an optional import later. | ADR-0003 |
| Q2 panel stack | Next.js App Router, three Vercel projects, subdomains. | ADR-0004 |
| Q3 variants | Yes — `ProductVariant` is the sellable unit, inventory keys on it. | ADR-0005 |
| Q14 registry | Moot — no shared packages. | ADR-0007 |
| Q15 app shares contracts | Moot — the app reads `API.md` like everything else. | ADR-0007 |
| Q16 spec location | `nexa-docs`, alongside `API.md`. | ADR-0007 |
| Q11 media & storage | Cloudinary, signed uploads. | ADR-0011 |
| Q8 notifications | Deferred until v1 parity is reached. | ADR-0012 |
| Q10 shipping & fulfilment | Deferred until v1 parity is reached. | ADR-0012 |
| Q4 credit terms | Interest-free, on-demand repayment, fixed admin-set limit, oldest-first allocation, overdue blocks new orders, distributors only. | ADR-0013 |
| Q6 referrals | Attribution-only, no reward, distributor-refers-distributor. | ADR-0014 |
| Q7 background jobs | Keep host crontab + scripts, no BullMQ/Redis worker. | ADR-0015 |
| Q9 invoicing & tax | Required at launch, flat tax rate, NexaShopping is seller of record. | ADR-0016 |
| Q12 mobile app | v1 app stays live, needs a server compat layer — NOT YET SCOPED, v1 API shapes needed. | ADR-0017 |
| Q13 team & environments | Solo dev, no staging yet, no CI yet — revisit staging before cutover. | ADR-0018 |
| Q5 payments | PhonePe (vendored, no published pkg exists), COD modeled, admin pays out distributors, refunds deferred. | ADR-0019 |

---

## Still open
