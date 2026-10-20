# NexaShopping documentation guide

Read `README.md` first for the active repository map, deployment pointers, and the current diary behavior. The older architecture, roadmap, schema draft, and phase plans preserve historical decisions; verify implementation claims in the active application repositories.

When changing an API, compare `docs/API.md` with the server route, schema, service, and controller and update the relevant contract section. For customer mobile routes, also check `nexa-server/docs/openapi/customer-mobile.yaml`. Keep money as decimal strings and distinguish cash diary records from shopping payments and credit.

Do not commit credentials, production `.env` files, OTPs, customer data, or generated build artifacts. Date any new status snapshot and state what was actually verified.
